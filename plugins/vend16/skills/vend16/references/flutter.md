# Flutter (in_app_purchase)

```bash
flutter pub add in_app_purchase in_app_purchase_storekit in_app_purchase_android http
```

Vend16 needs the StoreKit 2 **JWS** on iOS, so turn StoreKit 2 on before anything else touches purchases (it needs a recent `in_app_purchase_storekit`; with StoreKit 2 on, `serverVerificationData` is the JWS). On Android `serverVerificationData` is the purchase token.

Put the public key in `lib/vend16_config.dart` (the CLI does this).

## lib/vend16.dart

```dart
import 'dart:convert';
import 'dart:io' show Platform;
import 'package:http/http.dart' as http;
import 'package:in_app_purchase/in_app_purchase.dart';
import 'vend16_config.dart';

enum Outcome { settled, retryLater, rejected }

const _base = 'https://vend16.com';

/// POST /v1/receipts. productType: consumable, non_consumable or subscription.
Future<Outcome> reportToVend16(PurchaseDetails p, String userId, String productType, {int? priceMicros, String? currency}) async {
  try {
    final res = await http.post(
      Uri.parse('$_base/v1/receipts'),
      headers: {'Authorization': 'Bearer $vend16PublicKey', 'Content-Type': 'application/json'},
      body: jsonEncode({
        'platform': Platform.isIOS ? 'ios' : 'android',
        'token': p.verificationData.serverVerificationData,
        'productId': p.productID,
        'productType': productType,
        'appUserId': userId,
        if (priceMicros != null && currency != null) 'price': {'micros': priceMicros, 'currency': currency},
      }),
    ).timeout(const Duration(seconds: 30));
    if (res.statusCode == 422) return Outcome.rejected;
    if (res.statusCode < 200 || res.statusCode >= 300) return Outcome.retryLater;
    final status = jsonDecode(res.body)['status'];
    return const ['recorded', 'duplicate', 'revoked'].contains(status) ? Outcome.settled : Outcome.retryLater;
  } catch (_) {
    return Outcome.retryLater;
  }
}

/// GET /v1/subscribers/:id -> the product IDs this user can use right now.
Future<Set<String>> activeProductIds(String userId) async {
  final res = await http.get(Uri.parse('$_base/v1/subscribers/${Uri.encodeComponent(userId)}'),
      headers: {'Authorization': 'Bearer $vend16PublicKey'});
  if (res.statusCode != 200) throw Exception('Vend16 ${res.statusCode}');
  return Set<String>.from(jsonDecode(res.body)['active_product_ids'] as List);
}
```

## lib/store.dart

```dart
import 'dart:async';
import 'dart:io' show Platform;
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:in_app_purchase_android/in_app_purchase_android.dart';
import 'package:in_app_purchase_storekit/in_app_purchase_storekit.dart';
import 'vend16.dart';

const productTypes = {'coins_100': 'consumable', 'remove_ads': 'non_consumable', 'pro_monthly': 'subscription'};

class Store {
  Store(this.userId, this.onUnlocked);
  final String userId;
  final void Function(Set<String>) onUnlocked;
  final _iap = InAppPurchase.instance;
  StreamSubscription<List<PurchaseDetails>>? _sub;
  final products = <String, ProductDetails>{};

  /// Call once at launch, when the signed-in user is known.
  Future<void> start() async {
    if (Platform.isIOS) InAppPurchaseStoreKitPlatform.enableStoreKit2(); // JWS tokens for Vend16
    _sub = _iap.purchaseStream.listen((list) { for (final p in list) _handle(p); }); // listen first
    if (!await _iap.isAvailable()) return;
    final r = await _iap.queryProductDetails(productTypes.keys.toSet());
    for (final d in r.productDetails) { products[d.id] = d; } // show d.price, never a hard-coded price
    onUnlocked(await activeProductIds(userId).catchError((_) => <String>{}));
  }

  Future<void> buy(String productId) async {
    final d = products[productId];
    if (d == null) return;
    final param = PurchaseParam(productDetails: d, applicationUserName: userId);
    if (productTypes[productId] == 'consumable') {
      // autoConsume false: consume only after Vend16 has recorded it.
      await _iap.buyConsumable(purchaseParam: param, autoConsume: false);
    } else {
      await _iap.buyNonConsumable(purchaseParam: param); // subscriptions too
    }
  }

  Future<void> restore() => _iap.restorePurchases(); // restored purchases arrive on the stream

  Future<void> _handle(PurchaseDetails p) async {
    if (p.status != PurchaseStatus.purchased && p.status != PurchaseStatus.restored) return; // pending, error, canceled
    final type = productTypes[p.productID] ?? 'consumable';
    final d = products[p.productID];
    final outcome = await reportToVend16(p, userId, type,
        priceMicros: d == null ? null : (d.rawPrice * 1000000).round(), currency: d?.currencyCode);
    if (outcome == Outcome.retryLater) return; // not completed: the store redelivers it
    if (outcome == Outcome.settled && type == 'consumable' && Platform.isAndroid) {
      await _iap.getPlatformAddition<InAppPurchaseAndroidPlatformAddition>().consumePurchase(p);
    }
    if (p.pendingCompletePurchase) await _iap.completePurchase(p); // finish (iOS) / acknowledge (Android)
    if (outcome == Outcome.settled) onUnlocked(await activeProductIds(userId));
  }

  void dispose() => _sub?.cancel();
}
```

`applicationUserName` should be a UUID you keep per user (StoreKit 2 passes it to Apple as the appAccountToken, which must be a UUID), never an email. Gate features on the set `onUnlocked` gives you, never on a local flag.
