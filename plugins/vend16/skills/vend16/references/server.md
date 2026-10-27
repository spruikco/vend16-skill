# The app's own server (optional)

Apps without a backend do not need any of this: the app reports purchases with the public key and asks `GET /v1/subscribers/:appUserId` what to unlock.

If the app has a server, it can hold the secret key and react to purchases, renewals and refunds as they happen.

## Secret key

- Only in the server's environment, as `VEND16_SECRET_KEY`, in a git-ignored env file or the host's secret store. The CLI writes it there with `--server-env`; the `create_secret_key` tool returns it once (making a new one stops the old one).
- Never in app code, the repo, logs or chat.
- Secret-key endpoints: `POST /v1/receipts` (report from the server instead of the app), `GET /v1/purchases?app_user_id=&limit=`, `GET /v1/events?since=`, `GET /v1/metrics?days=`, `DELETE /v1/subscribers/:appUserId` (privacy deletion).

## Webhooks

Set the URL on the Setup page (or the `set_webhook` tool), then `send_test_webhook`.

Events: `initial_purchase`, `renewal`, `non_subscription_purchase`, `cancellation`, `uncancellation`, `billing_issue`, `billing_recovered`, `paused`, `expiration`, `refund`, `revocation`, `refund_reversed`.

Every request carries `Vend16-Signature: t=<unix>,v1=<hex>`, where hex is HMAC-SHA256 of `t + "." + rawBody` with the webhook secret. Verify against the **raw** body, before parsing:

```js
import { createHmac, timingSafeEqual } from 'node:crypto';

export function verifyVend16(rawBody, header, secret) {
  const { t, v1 } = Object.fromEntries(String(header).split(',').map((p) => p.split('=')));
  if (!t || !v1 || Math.abs(Date.now() / 1000 - Number(t)) > 300) return false;
  const expected = createHmac('sha256', secret).update(t + '.' + rawBody).digest('hex');
  return v1.length === expected.length && timingSafeEqual(Buffer.from(v1), Buffer.from(expected));
}
```

Next.js route:

```ts
// app/api/webhooks/vend16/route.ts
import { verifyVend16 } from '@/lib/vend16';

export async function POST(req: Request) {
  const raw = await req.text();
  if (!verifyVend16(raw, req.headers.get('vend16-signature'), process.env.VEND16_WEBHOOK_SECRET!)) {
    return new Response('bad signature', { status: 401 });
  }
  const event = JSON.parse(raw);
  // Idempotent: the same event.id can arrive more than once.
  // e.g. on 'refund' or 'revocation', take back what event.data granted to event.app_user_id.
  return new Response('ok');
}
```

Reply 2xx quickly; Vend16 retries failures for about three days.
