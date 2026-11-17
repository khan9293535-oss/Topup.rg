# Nocturne: deployment and operator notes

This upgrade preserves the existing catalog, accounts, orders and URLs. Nocturne is an independent marketplace; adding a provider account does **not** establish authorization from any publisher. Confirm the contract, permitted regions and each product/denomination before mapping it.

## Safe defaults

- With no credentials, products are catalog previews. Signed-in customers may save an **unpaid request**. No money is taken and no provider order is sent.
- Existing historical `test` orders are retained and identified as legacy simulations. They are not evidence of payment or delivery.
- Prices in the original seed catalog are indicative, not verified market prices. Review all products, player field formats, regions, retail prices, and provider costs before selling.
- Easypaisa, JazzCash, and bank transfer are visibly unavailable. Their merchant approval and a properly verified integration are still required. Stripe merchant availability depends on your business's country; Pakistan is not currently a Stripe-supported merchant country. Do not offer an unsupported method as active.

## Environment variables (server only)

Keep secrets out of source control and the browser. Set these in the hosting environment:

- `DATABASE_URL`: PostgreSQL URL (already provided in the development sandbox).
- `ADMIN_EMAILS`: comma-separated emails eligible for operator setup.
- `ADMIN_BOOTSTRAP_TOKEN`: a long, random secret of at least 24 characters. A signed-in eligible user visits `/operator-setup` and provides this token. Existing database admins retain access. The first public signup is **not** an admin. Remove or rotate the token after setup.
- `PROVIDER_ENCRYPTION_KEY`: exactly 64 hexadecimal characters (32 random bytes). Example generation command: `openssl rand -hex 32`. Preserve this key securely; changing or losing it makes stored provider credentials and gift-card codes unreadable.
- `STRIPE_SECRET_KEY`: official Stripe secret key (`sk_test_…` or `sk_live_…`).
- `STRIPE_WEBHOOK_SECRET`: signing secret for the endpoint below.
- `APP_URL`: canonical public origin, e.g. `https://your-domain.example` (no path or trailing slash required).
- `CRON_SECRET`: random secret for the scheduled reconciliation endpoint.

No secret needs a `NEXT_PUBLIC_` prefix. Do not put real keys in this file.

## Payments

Register `https://YOUR-DOMAIN/api/webhooks/stripe` in Stripe and subscribe to `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, `checkout.session.expired`, and `charge.refunded`. Use the correct test or live webhook secret. Hosted checkout is created with an idempotency key, and a signed webhook causes the server to retrieve the session from Stripe and verify its amount, currency, reference and payment status. Returning from checkout alone does not mark an order paid.

Test-mode Stripe payments are labeled as such and **never** dispatch a real top-up. Test thoroughly with Stripe CLI and test credentials before enabling live payments. Only failed, verified live payments can be refunded from the delivery desk; a refund remains pending until Stripe confirms it. An uncertain external refund should be reconciled in the Stripe dashboard rather than submitted twice.

References: https://docs.stripe.com/checkout/fulfillment and https://docs.stripe.com/webhooks

## Providers and verification

From Delivery desk → Providers & payments, add and test an authorized GamesDrop shop token or Reloadly Gift Cards client ID/secret. Provider credentials are encrypted with AES-256-GCM. Multiple connections can be added without rebuilding. Sandbox Reloadly connections never make catalog packages live. Map each package to its actual partner offer/product ID after checking the product, denomination, required fields, region and costs. GamesDrop test offer 999 cannot be sold. Reloadly automatic delivery is intentionally restricted to USD sender/recipient gift cards without extra user-ID requirements; other configurations remain unavailable until explicitly supported.

For GamesDrop products where the partner supports it, enable required UID verification in the mapping and identify the correct field keys. The checkout displays only the actual provider response; a valid response without a nickname is shown without inventing one. Validation is repeated on the server before payment and again before dispatch. If a provider is unavailable, verification cannot be bypassed for a mapped product requiring it.

Live paid orders are submitted with their original unique Nocturne reference. Uncertain submission outcomes remain in progress and are reconciled with that same reference, never automatically resubmitted as a new purchase. Gift-card redemption details are encrypted and shown only on the authorized customer's order page. Schedule a GET to `/api/cron/orders` with `Authorization: Bearer <CRON_SECRET>` periodically; open order pages also refresh status. Monitor provider balances, API errors and orders that need manual review.

References: https://gamesdrop.io/en/docs/partner-api and https://docs.reloadly.com/gift-cards

## Database and validation

After the platform runtime is bootstrapped, run `npx drizzle-kit push` to apply the additive schema. Back up production data and inspect changes before applying them. Validate with `npx next typegen`, `npm exec tsc -- --noEmit --pretty false`, and `npm run build`.
