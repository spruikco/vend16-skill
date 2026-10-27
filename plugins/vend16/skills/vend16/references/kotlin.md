# Kotlin (Google Play Billing)

Add Play Billing to `app/build.gradle(.kts)` if it is not there:

```kotlin
implementation("com.android.billingclient:billing-ktx:7.1.1")
```

Put the public key in `res/values/vend16.xml` as `vend16_public_key` (the CLI does this).

Vend16 acknowledges non-consumables and subscriptions on its server, so Google never auto-refunds them. The app only **consumes consumables**, and only after Vend16 has recorded the purchase.

## Vend16.kt

```kotlin
import android.content.Context
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URL
import java.net.URLEncoder

/** Talks to Vend16. Only the public key (pk_...) belongs in the app. */
object Vend16 {
    private const val BASE = "https://vend16.com"

    enum class Outcome { SETTLED, RETRY_LATER, REJECTED }

    /**
     * POST /v1/receipts. SETTLED: Vend16 has it. RETRY_LATER: change nothing,
     * Play redelivers it. REJECTED: 422, Google says it is not real; unlock nothing.
     * productType: "consumable", "non_consumable" or "subscription" (Google does not say).
     */
    suspend fun report(
        context: Context, purchaseToken: String, productId: String, productType: String,
        userId: String, priceMicros: Long?, currency: String?,
    ): Outcome = withContext(Dispatchers.IO) {
        val body = JSONObject()
            .put("platform", "android")
            .put("token", purchaseToken)
            .put("productId", productId)
            .put("productType", productType)
            .put("appUserId", userId)
        if (priceMicros != null && currency != null) body.put("price", JSONObject().put("micros", priceMicros).put("currency", currency))
        try {
            val c = (URL("$BASE/v1/receipts").openConnection() as HttpURLConnection).apply {
                requestMethod = "POST"; doOutput = true; connectTimeout = 15_000; readTimeout = 30_000
                setRequestProperty("Authorization", "Bearer ${key(context)}")
                setRequestProperty("Content-Type", "application/json")
            }
            c.outputStream.use { it.write(body.toString().toByteArray()) }
            when (c.responseCode) {
                in 200..299 -> {
                    val status = JSONObject(c.inputStream.bufferedReader().readText()).optString("status")
                    if (status in setOf("recorded", "duplicate", "revoked")) Outcome.SETTLED else Outcome.RETRY_LATER
                }
                422 -> Outcome.REJECTED
                else -> Outcome.RETRY_LATER
            }
        } catch (e: Exception) {
            Outcome.RETRY_LATER
        }
    }

    /** GET /v1/subscribers/:id -> the product IDs this user can use right now. */
    suspend fun activeProductIds(context: Context, userId: String): Set<String> = withContext(Dispatchers.IO) {
        val c = URL("$BASE/v1/subscribers/${URLEncoder.encode(userId, "UTF-8")}").openConnection() as HttpURLConnection
        c.setRequestProperty("Authorization", "Bearer ${key(context)}")
        val arr = JSONObject(c.inputStream.bufferedReader().readText()).optJSONArray("active_product_ids")
        (0 until (arr?.length() ?: 0)).map { arr!!.getString(it) }.toSet()
    }

    private fun key(context: Context) = context.getString(R.string.vend16_public_key)
}
```

## Store.kt

```kotlin
import android.app.Activity
import android.content.Context
import com.android.billingclient.api.*
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.launch

class Store(
    private val context: Context,
    private val scope: CoroutineScope,
    private val userId: String,
    /** productId -> "consumable" | "non_consumable" | "subscription" */
    private val types: Map<String, String>,
    private val onUnlocked: (Set<String>) -> Unit,
) : PurchasesUpdatedListener {
    private val details = mutableMapOf<String, ProductDetails>()
    private val client = BillingClient.newBuilder(context)
        .setListener(this)
        .enablePendingPurchases(PendingPurchasesParams.newBuilder().enableOneTimeProducts().build())
        .build()

    fun start() = client.startConnection(object : BillingClientStateListener {
        override fun onBillingSetupFinished(result: BillingResult) {
            if (result.responseCode != BillingClient.BillingResponseCode.OK) return
            scope.launch {
                loadDetails()
                // Purchases Play still holds (unconsumed, or bought while the app was closed).
                for (type in listOf(BillingClient.ProductType.INAPP, BillingClient.ProductType.SUBS)) {
                    client.queryPurchasesAsync(QueryPurchasesParams.newBuilder().setProductType(type).build())
                        .purchasesList.forEach { handle(it) }
                }
                refresh()
            }
        }
        override fun onBillingServiceDisconnected() { /* retry start() on the next store visit */ }
    })

    private suspend fun loadDetails() {
        for ((type, ids) in types.keys.groupBy { if (types[it] == "subscription") BillingClient.ProductType.SUBS else BillingClient.ProductType.INAPP }) {
            val params = QueryProductDetailsParams.newBuilder().setProductList(ids.map {
                QueryProductDetailsParams.Product.newBuilder().setProductId(it).setProductType(type).build()
            }).build()
            client.queryProductDetails(params).productDetailsList?.forEach { details[it.productId] = it }
        }
    }

    /** Show prices from here (localised by Play), never hard-coded ones. */
    fun product(id: String): ProductDetails? = details[id]

    fun buy(activity: Activity, productId: String, accountId: String) {
        val d = details[productId] ?: return
        val pp = BillingFlowParams.ProductDetailsParams.newBuilder().setProductDetails(d)
        d.subscriptionOfferDetails?.firstOrNull()?.let { pp.setOfferToken(it.offerToken) }
        client.launchBillingFlow(activity, BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(listOf(pp.build()))
            .setObfuscatedAccountId(accountId) // a hash or UUID of the user, never an email
            .build())
    }

    override fun onPurchasesUpdated(result: BillingResult, purchases: MutableList<Purchase>?) {
        if (result.responseCode == BillingClient.BillingResponseCode.OK) purchases?.forEach { scope.launch { handle(it) } }
    }

    private suspend fun handle(p: Purchase) {
        if (p.purchaseState != Purchase.PurchaseState.PURCHASED) return // PENDING: wait for Play to call again
        val productId = p.products.first()
        val type = types[productId] ?: "consumable"
        val d = details[productId]
        val micros = d?.oneTimePurchaseOfferDetails?.priceAmountMicros
            ?: d?.subscriptionOfferDetails?.firstOrNull()?.pricingPhases?.pricingPhaseList?.lastOrNull()?.priceAmountMicros
        val currency = d?.oneTimePurchaseOfferDetails?.priceCurrencyCode
            ?: d?.subscriptionOfferDetails?.firstOrNull()?.pricingPhases?.pricingPhaseList?.lastOrNull()?.priceCurrencyCode
        when (Vend16.report(context, p.purchaseToken, productId, type, userId, micros, currency)) {
            Vend16.Outcome.SETTLED -> {
                // Vend16 acknowledged non-consumables and subscriptions server-side.
                if (type == "consumable") client.consumePurchase(ConsumeParams.newBuilder().setPurchaseToken(p.purchaseToken).build())
                refresh()
            }
            Vend16.Outcome.REJECTED -> Unit // not real: unlock nothing
            Vend16.Outcome.RETRY_LATER -> Unit // change nothing; Play keeps it until the next try
        }
    }

    suspend fun refresh() = runCatching { onUnlocked(Vend16.activeProductIds(context, userId)) }
}
```

Restore on Android is `start()` again: `queryPurchasesAsync` returns everything the user owns, and each is reported (Vend16 answers `duplicate`). Gate features on the set `onUnlocked` gives you, never on a local flag.
