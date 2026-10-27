---
name: vend16
description: Add in-app purchases and subscriptions to a mobile app with Vend16, which verifies every App Store and Google Play purchase server-side, tracks renewals and refunds, and tells the app what the user has paid for. Use when the user wants to add in-app purchases, subscriptions, a paywall, StoreKit 2, Google Play Billing, receipt validation, or a RevenueCat alternative to a Capacitor, React Native, Expo, Flutter, Swift or Kotlin app.
---

# Vend16: add verified in-app purchases

Vend16 is a hosted backend for in-app purchases. The app hands each store purchase to Vend16, Vend16 checks it with Apple or Google, and the app unlocks features from Vend16's answer instead of trusting the device. Put a buy button in the app; Vend16 gets the rest sorted.

Your job: get the user a Vend16 app and key with as little typing from them as possible, wire purchases into their code following the money rules, and finish with an exact list of what only they can click in the store consoles.

## Step 1: read the current guide

Fetch `https://vend16.com/llms-full.txt`. It is the source of truth for endpoints, fields and error codes. This skill describes the workflow; the references in this folder hold copy-ready code per platform.

## Step 2: work out the app

Look at the repo:

1. **Platform.** Capacitor (`capacitor.config.*`), Expo (`expo` in package.json), React Native, Flutter (`pubspec.yaml`), native iOS (`.xcodeproj`), native Android (`app/build.gradle`).
2. **Store IDs.** The iOS bundle ID and the Android application ID.
3. **Own backend?** If the app has a server, it can hold the secret key and receive webhooks. If not, the app talks to Vend16 directly with the public key. Both are fine.
4. **Products.** Find the product IDs (paywall, config, constants). If there are none, ask the user for each ID and whether it is a consumable, a non-consumable or a subscription.
5. **User ID.** The app's stable user ID (the auth user's ID). Never an email address.

## Step 3: account and keys (the user should not have to copy anything)

Use the first path that works.

**A. Vend16 tools (MCP).** If you have tools named `whoami`, `list_apps`, `create_app`, `get_app_keys`, `setup_status`, use them. This plugin adds the Vend16 MCP server (`https://vend16.com/mcp`); the first call opens a browser sign-in for the user. Then:

1. `whoami` to confirm the account.
2. `list_apps`. If no app has this bundle ID or package name, `create_app` with the app's name, bundle ID and package name.
3. `get_app_keys` for the public key (`pk_...`).
4. Only if the app has its own server: `create_secret_key`, and write the result straight into the server's git-ignored env file as `VEND16_SECRET_KEY`. Do not echo it in your reply.

**B. The CLI.** If there are no Vend16 tools, run this in the app's root:

```bash
npx github:spruikco/vend16-cli init --yes --json
```

It prints a short sign-in code to stderr and opens https://vend16.com/activate. Show the user the code and ask them to approve it; the command then finds or creates the app, writes the public key where the app expects it, installs the store client and prints JSON with `public_key.read_with`, `setup`, `manual` and `next`. Add `--server-env path/to/.env` if the app has a server. Exit 4 with `candidates` means pick an app with `--app app_...`.

*(Paths A and B depend on the Vend16 MCP server and CLI sign-in, which are rolling out on vend16.com. If either answers "not available" or "not switched on", use path C.)*

**C. By hand.** Ask the user:

> Create a free account at https://vend16.com/signup, add an app with your iOS bundle ID and Android package name, and paste the public key (pk_...) here. Free covers your first 100 live purchases; test purchases are unlimited.

Key rules on every path:

- The public key (`pk_...`) is safe in app code. Put it in the app's usual public config, once.
- Never ask for, print, log or commit the secret key (`sk_...`). It only goes in a server's environment as `VEND16_SECRET_KEY`.

## Step 4: wire it in

Read the reference for the platform and follow it:

| Platform | Reference |
| --- | --- |
| Capacitor 8 | [references/capacitor.md](references/capacitor.md) |
| Swift (StoreKit 2) | [references/swift.md](references/swift.md) |
| Kotlin (Play Billing) | [references/kotlin.md](references/kotlin.md) |
| React Native, Expo | [references/react-native.md](references/react-native.md) |
| Flutter | [references/flutter.md](references/flutter.md) |
| The app's own server | [references/server.md](references/server.md) |

The money rules. Apply all of them, on every platform:

1. **Report, then finish.** Send every completed purchase to `POST /v1/receipts`. Call the store's finish, consume or acknowledge only after Vend16 replies `recorded` or `duplicate` (`revoked` too: it is settled).
2. **A network error or `502 store_unavailable` is not a failure.** Do not finish. The store redelivers the purchase on the next launch. Tell the user "We could not confirm your purchase yet", not "failed".
3. **`422 invalid_receipt`:** the store says it is not real. Finish it so it stops coming back, and unlock nothing.
4. **Unlock from the server.** Gate features on `active_product_ids` from `GET /v1/subscribers/:appUserId`. Refresh on app start, after a purchase and after restore. Never unlock from a local flag.
5. **Listen for purchases at launch,** before showing the store: both stores redeliver unfinished purchases then.
6. **Restore button** on the paywall (Apple requires one for non-consumables and subscriptions).
7. **Android: send `productType` and `price`.** Google does not say whether a one-off product is consumable, or what it cost.

## Step 5: store setup

Code alone does not take money. Read [references/store-setup.md](references/store-setup.md).

With Vend16 tools: `setup_status` returns the checklist with what is done ticked off; `upload_play_service_account` takes the Google service account JSON if the user gives you the file path; `set_webhook` and `send_test_webhook` set up and test a server's webhook; `list_recent_purchases` shows whether a sandbox purchase arrived. With the CLI: `npx github:spruikco/vend16-cli status`.

## Step 6: finish with a checklist

End your turn with exactly what is left that only the user can do, each with where to click. At minimum:

- Products created in App Store Connect and Play Console with the same IDs as the code.
- App Store Server Notifications (Version 2), Production and Sandbox, set to `https://vend16.com/v1/notifications/apple/<APP_ID>`.
- Google Play service account uploaded on the app's Setup page, and real-time developer notifications turned on.
- One sandbox purchase on each platform, then check it shows under Sandbox in the Vend16 dashboard.

Then run the app's tests or build if they are quick, and say which money rules you applied where.
