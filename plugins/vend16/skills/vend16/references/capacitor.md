# Capacitor 8

Use the open-source plugin `@spruik/capacitor-iap` (StoreKit 2 and Play Billing behind one interface). Its [AGENTS.md](https://github.com/spruikco/capacitor-iap/blob/main/AGENTS.md) is the full step-by-step; this is the short form.

Needs Capacitor 8 (`@capacitor/core` 8.x). On 7 or older, tell the user to upgrade first.

## Install

```bash
npm install github:spruikco/capacitor-iap
npx cap sync
```

`cap sync` does all the native wiring (Swift package on iOS, Gradle module on Android). Do not edit native project files.

## Code

One module, for example `src/purchases.ts`. Call `startPurchases(userId)` once, when the signed-in user is known.

```ts
import { SpruikIap } from '@spruik/capacitor-iap';
import { createVend16 } from '@spruik/capacitor-iap/vend16';

const vend16 = createVend16({
  apiKey: import.meta.env.VITE_VEND16_PUBLIC_KEY, // pk_..., or however the CLI said to read it
  productTypes: { pro_monthly: 'subscription', remove_ads: 'non_consumable', coins_100: 'consumable' },
});

export const PRODUCT_IDS = ['coins_100', 'remove_ads', 'pro_monthly'];

export async function startPurchases(userId: string) {
  // Listen BEFORE initialize: the stores replay unfinished purchases on connect.
  await SpruikIap.addListener('transactionUpdated', (tx) => vend16.handle(tx, userId, SpruikIap));
  return (await SpruikIap.initialize({ productIds: PRODUCT_IDS })).available;
}

export async function products() {
  return (await SpruikIap.getProducts()).products; // localised prices: show these, never hard-coded ones
}

export async function buy(productId: string, userId: string) {
  const { status, transaction } = await SpruikIap.purchase({ productId });
  if (status === 'purchased') await vend16.handle(transaction, userId, SpruikIap);
  return status; // 'cancelled' and 'pending' are normal
}

export async function unlocked(userId: string) {
  return new Set((await vend16.subscriber(userId)).active_product_ids);
}

export async function restore(userId: string) {
  // Restored purchases were finished long ago: report them, do not finish them again.
  const { transactions } = await SpruikIap.restore();
  for (const tx of transactions) await vend16.record(tx, userId).catch(() => {});
  return unlocked(userId);
}
```

`vend16.handle` already applies the money rules: it reports first, finishes only on `recorded`, `duplicate` or `revoked`, consumes consumables and acknowledges the rest, leaves a purchase unfinished on network errors and `502`, and finishes without consuming on `422`.

To report Android prices for revenue, pass the price from `getProducts()`: `vend16.handle(tx, userId, SpruikIap, { micros: product.priceMicros, currency: product.currency })`.
