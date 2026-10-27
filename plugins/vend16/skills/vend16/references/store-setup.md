# Store setup: what the user clicks

The code is half of it. Walk the user through these, in this order, with these exact places. `<APP_ID>` is the Vend16 app ID (`app_...`), shown on the app's Setup page at https://vend16.com/app.

With Vend16 tools, `setup_status` says which of these Vend16 already sees working. With the CLI: `npx github:spruikco/vend16-cli status`.

## App Store Connect (iOS)

1. **Paid Apps agreement.** Business (or Agreements, Tax and Banking): the Paid Apps agreement must be active, with banking and tax filled in. Without it no product loads, even in sandbox.
2. **Products.** Your app, Monetization:
   - In-App Purchases: Consumable or Non-Consumable, with the same product ID as the code.
   - Subscriptions: create a subscription group, then each subscription with the same ID as the code.
   - Each needs a price, a display name and a review screenshot before it can be submitted. The first products go to review with an app version.
3. **Server notifications.** Your app, App Information, App Store Server Notifications:
   - Production Server URL: `https://vend16.com/v1/notifications/apple/<APP_ID>`
   - Sandbox Server URL: the same URL
   - Version: **Version 2** for both
   This is how Vend16 hears about renewals, refunds and cancellations. No Apple API key is needed.
4. **Test.** Users and Access, Sandbox, Test Accounts: make a sandbox Apple Account. Install a TestFlight or Xcode build on a real iPhone (simulators cannot buy) and buy. It shows under Sandbox in the Vend16 dashboard within seconds.

## Google Play Console (Android)

1. **Payments profile** linked to the developer account (Settings, Payments profile).
2. **Upload a build** with Play Billing to a testing track (Internal testing is fine). Play only lets you create products once a billing-enabled build exists.
3. **Products.** Monetize with Play, Products: In-app products and Subscriptions, same IDs as the code. Activate each one.
4. **Service account, so Vend16 can check purchases.**
   1. Google Cloud console (any project): APIs and services, enable **Google Play Android Developer API**.
   2. IAM and admin, Service accounts, Create service account (no roles needed in Cloud). Keys, Add key, JSON: this downloads the key file.
   3. Play Console, Users and permissions, Invite new users: the service account's email. App permissions: your app, with **View financial data** and **Manage orders and subscriptions**. Account permissions can stay empty.
   4. Upload the JSON key on the app's Setup page at vend16.com (or the `upload_play_service_account` tool with the file's path). Never commit it, never paste it into chat.
   New service accounts can take up to a day before Google accepts them. Until then Vend16 answers `502 store_unavailable`, and the app correctly leaves purchases unfinished.
5. **Real-time developer notifications (renewals and refunds).**
   1. Google Cloud console, Pub/Sub, Create topic (for example `play-rtdn`).
   2. The topic, Permissions, Add principal `google-play-developer-notifications@system.gserviceaccount.com` with role **Pub/Sub Publisher**.
   3. The topic, Create subscription, delivery type **Push**, endpoint: the Play notifications URL shown on the Vend16 Setup page.
   4. Play Console, Monetize with Play, Monetization setup: Real-time developer notifications, paste the full topic name (`projects/<project>/topics/play-rtdn`), Send test notification. The Setup page ticks once it arrives.
6. **Test.** Settings, License testing: add the tester's Google account. Install from the testing track's opt-in link and buy with a test card. It shows under Sandbox in the Vend16 dashboard.

## Vend16 (the app's Setup page)

- iOS bundle ID and Android package name match the app.
- A webhook URL, only if the app has its own server (see server.md).
- Live or Sandbox toggle in the dashboard: test purchases are under Sandbox.
