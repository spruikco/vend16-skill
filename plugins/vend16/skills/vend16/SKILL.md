---
name: vend16
description: Add in-app purchases and subscriptions to a mobile app with Vend16, which verifies every App Store and Google Play purchase server-side, tracks renewals and refunds, and tells the app what the user has paid for. Use when the user wants to add in-app purchases, subscriptions, a paywall, StoreKit 2, Google Play Billing, receipt validation, or a RevenueCat alternative to a Capacitor, React Native, Flutter, Swift or Kotlin app.
---

# Vend16: add verified in-app purchases

Vend16 is a hosted backend for in-app purchases. The app hands each store purchase to Vend16, Vend16 checks it with Apple or Google, and the app unlocks features from Vend16's answer instead of trusting the device.

## Step 1: read the current guide

Fetch `https://vend16.com/llms-full.txt` and treat it as the source of truth for endpoints, field names and code. It is kept current; this skill only describes the workflow.

## Step 2: work out the app

Look at the repo and decide:

1. **Platform.** Capacitor (`capacitor.config.*`), React Native, Flutter, native iOS (Swift) or native Android (Kotlin).
2. **Has its own backend?** If there is a server, purchases can be reported with the secret key from the server, and webhooks can be used. If not, the app talks to Vend16 directly with the public key. Both are supported.
3. **Products.** Find existing product IDs (App Store Connect / Play Console IDs in code, config or a paywall). If none exist, ask the user for them and whether each is a consumable, non-consumable or subscription.
4. **User ID.** Find the app's stable user ID (auth user id, not an email). Purchases are attached to it.

## Step 3: keys

The user needs a free Vend16 account. If they have not given you a public key (`pk_...`), tell them:

> Create a free account at https://vend16.com/signup, create an app with your iOS bundle ID and Android package name, and paste the public key here. Free covers your first 100 live purchases; test purchases are unlimited.

Never ask for, print or commit the secret key (`sk_...`). If the app has a server, put it in that server's environment as `VEND16_SECRET_KEY`.

## Step 4: wire it in

- **Capacitor:** install `npm install github:spruikco/capacitor-iap` and `npx cap sync`, then use the `createVend16` helper from `@spruik/capacitor-iap/vend16` exactly as `llms-full.txt` shows. Register the `transactionUpdated` listener before `initialize`.
- **Anything else:** after each completed store purchase, `POST /v1/receipts` with the StoreKit 2 JWS (iOS) or the Play purchase token plus product ID (Android).

Rules that matter for money, apply all of them:

1. Only call the store's finish / consume / acknowledge after Vend16 replies with `recorded` or `duplicate`. On `502 store_unavailable` or a network error, do not finish: the store redelivers it.
2. On `422 invalid_receipt`, finish it and unlock nothing.
3. Unlock features from `GET /v1/subscribers/:appUserId` (`active_product_ids`), never from local flags. Refresh it on app start and after purchases.
4. Add a restore purchases action.
5. On Android, send the price (`micros` and `currency`) so revenue reports are correct.

If the app has a backend, add a webhook endpoint that verifies the `Vend16-Signature` header as shown in `llms-full.txt`, and update the user's access from the events.

## Step 5: finish with a checklist

End by telling the user exactly what is left that only they can do:

- App Store Connect, App Information, App Store Server Notifications: set Production and Sandbox URLs (Version 2) to the URL shown on the app's Setup page at vend16.com.
- Google Play: create the service account, invite it in Play Console with financial and order permissions, paste its JSON on the Setup page, and set up the Pub/Sub topic for real-time notifications (steps in `llms-full.txt`).
- Make one sandbox purchase on each platform and confirm it appears in the Vend16 dashboard under Sandbox.
