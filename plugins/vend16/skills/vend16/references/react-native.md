# React Native and Expo (react-native-iap)

```bash
npm install react-native-iap
cd ios && pod install   # bare React Native, on a Mac
```

Expo: react-native-iap needs a development build (`npx expo prebuild`, or EAS Build). Expo Go cannot make purchases.

Written for **react-native-iap 14** (StoreKit 2 on iOS). If the installed major version is older, check its README for the same calls; the logic does not change. Vend16 needs the StoreKit 2 **JWS** on iOS. The legacy base64 receipt (`transactionReceipt`) is not accepted, so use a version that gives the JWS.

Put the public key where the CLI said (`vend16.config.js`, or Expo's `extra.vend16PublicKey`).

## vend16.ts

```ts
import { Platform } from 'react-native';
import { VEND16_PUBLIC_KEY } from './vend16.config';
// Expo: import Constants from 'expo-constants'; const VEND16_PUBLIC_KEY = Constants.expoConfig?.extra?.vend16PublicKey;

const BASE = 'https://vend16.com';
type Outcome = 'settled' | 'retryLater' | 'rejected';
export type ProductType = 'consumable' | 'non_consumable' | 'subscription';

/** The token Vend16 checks: the StoreKit 2 JWS on iOS, the purchase token on Android. */
function storeToken(p: any): string | undefined {
  return Platform.OS === 'ios' ? p.jwsRepresentationIOS ?? p.jwsRepresentationIos ?? p.purchaseToken : p.purchaseToken ?? p.purchaseTokenAndroid;
}

export async function report(p: any, userId: string, type: ProductType, price?: { micros: number; currency: string }): Promise<Outcome> {
  const token = storeToken(p);
  if (!token) return 'retryLater';
  try {
    const res = await fetch(`${BASE}/v1/receipts`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${VEND16_PUBLIC_KEY}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({
        platform: Platform.OS === 'ios' ? 'ios' : 'android',
        token,
        productId: p.productId ?? p.id,
        productType: type, // Google does not say; Apple ignores it
        appUserId: userId,
        price, // Android revenue reporting
      }),
    });
    if (res.status === 422) return 'rejected';
    if (!res.ok) return 'retryLater';
    const { status } = await res.json();
    return ['recorded', 'duplicate', 'revoked'].includes(status) ? 'settled' : 'retryLater';
  } catch {
    return 'retryLater';
  }
}

export async function activeProductIds(userId: string): Promise<Set<string>> {
  const res = await fetch(`${BASE}/v1/subscribers/${encodeURIComponent(userId)}`, {
    headers: { Authorization: `Bearer ${VEND16_PUBLIC_KEY}` },
  });
  if (!res.ok) throw new Error(`Vend16 ${res.status}`);
  return new Set((await res.json()).active_product_ids);
}
```

## purchases.ts

```ts
import {
  initConnection, fetchProducts, requestPurchase, finishTransaction, getAvailablePurchases,
  purchaseUpdatedListener, purchaseErrorListener,
} from 'react-native-iap';
import { report, activeProductIds, type ProductType } from './vend16';

export const TYPES: Record<string, ProductType> = { coins_100: 'consumable', remove_ads: 'non_consumable', pro_monthly: 'subscription' };

let subs: Array<{ remove: () => void }> = [];

/** Call once when the signed-in user is known. */
export async function startPurchases(userId: string, onUnlocked: (ids: Set<string>) => void) {
  await initConnection();
  // Listen first: unfinished purchases are redelivered here.
  subs.push(purchaseUpdatedListener(async (purchase) => {
    const id = purchase.productId ?? (purchase as any).id;
    const type = TYPES[id] ?? 'consumable';
    const outcome = await report(purchase, userId, type);
    if (outcome === 'settled') {
      await finishTransaction({ purchase, isConsumable: type === 'consumable' });
      onUnlocked(await activeProductIds(userId));
    } else if (outcome === 'rejected') {
      await finishTransaction({ purchase, isConsumable: false }); // not real: unlock nothing
    }
    // 'retryLater': leave it unfinished; the store redelivers it.
  }));
  subs.push(purchaseErrorListener(() => { /* user cancelled or store error: nothing to undo */ }));
  onUnlocked(await activeProductIds(userId).catch(() => new Set<string>()));
}

export function stopPurchases() { subs.forEach((s) => s.remove()); subs = []; }

export async function loadProducts() {
  const inApp = Object.keys(TYPES).filter((id) => TYPES[id] !== 'subscription');
  const subsIds = Object.keys(TYPES).filter((id) => TYPES[id] === 'subscription');
  return [
    ...(inApp.length ? await fetchProducts({ skus: inApp, type: 'in-app' }) : []),
    ...(subsIds.length ? await fetchProducts({ skus: subsIds, type: 'subs' }) : []),
  ]; // show their localised prices, never hard-coded ones
}

export async function buy(productId: string, accountToken: string) {
  // The result arrives in purchaseUpdatedListener, which reports it.
  await requestPurchase({
    request: { ios: { sku: productId, appAccountToken: accountToken }, android: { skus: [productId], obfuscatedAccountIdAndroid: accountToken } },
    type: TYPES[productId] === 'subscription' ? 'subs' : 'in-app',
  });
}

/** Restore button: report what the stores say the user owns, then ask Vend16. */
export async function restore(userId: string) {
  for (const p of await getAvailablePurchases()) {
    const id = p.productId ?? (p as any).id;
    await report(p, userId, TYPES[id] ?? 'non_consumable');
  }
  return activeProductIds(userId);
}
```

`accountToken` is a UUID you keep per user (Apple requires a UUID for `appAccountToken`). Gate features on the set from `activeProductIds`, never on a local flag.
