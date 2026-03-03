---
id: stripe
title: Stripe Integration
sidebar_position: 3
description: Production Stripe billing setup for BlackRoad OS — subscription management, webhooks, and Converter API access gating.
---

# Stripe Integration

> **Status:** Production  
> **Owner:** @blackboxprogramming

BlackRoad OS uses Stripe to manage subscriptions for access to the Converter API and the BlackRoad OS platform. An active subscription is required to obtain and refresh Converter API tokens.

---

## Products & Pricing

| Product | Monthly Price | Converter API Limit | Features |
|---------|--------------|---------------------|----------|
| **Basic** | $9.00 / mo | 1,000 req/day | Lucidia backend, governance layer |
| **Pro** | $29.00 / mo | 10,000 req/day | All backends, priority routing |
| **Enterprise** | $99.00 / mo | Unlimited | Dedicated nodes, SLA, custom agents |

These products are defined in `stripe-config.json` at the root of this repository.

---

## Setup

### 1. Install the Stripe CLI (for local testing)

```bash
brew install stripe/stripe-cli/stripe
stripe login
```

### 2. Configure environment variables

```bash
# .env (never commit this file)
STRIPE_SECRET_KEY=<your-stripe-live-secret-key>
STRIPE_WEBHOOK_SECRET=<your-stripe-webhook-secret>
STRIPE_PUBLISHABLE_KEY=<your-stripe-publishable-key>
```

For local development use test keys:

```bash
STRIPE_SECRET_KEY=<your-stripe-test-secret-key>
STRIPE_WEBHOOK_SECRET=<your-stripe-test-webhook-secret>
```

### 3. Create products in Stripe

Use the Stripe CLI or dashboard to create the products defined in `stripe-config.json`:

```bash
# Basic — $9/month
stripe products create --name="BlackRoad OS - Basic"
stripe prices create \
  --unit-amount=900 \
  --currency=usd \
  --recurring[interval]=month \
  --product=PRODUCT_ID

# Pro — $29/month
stripe products create --name="BlackRoad OS - Pro"
stripe prices create \
  --unit-amount=2900 \
  --currency=usd \
  --recurring[interval]=month \
  --product=PRODUCT_ID

# Enterprise — $99/month
stripe products create --name="BlackRoad OS - Enterprise"
stripe prices create \
  --unit-amount=9900 \
  --currency=usd \
  --recurring[interval]=month \
  --product=PRODUCT_ID
```

---

## Checkout Flow

Customers subscribe by completing a Stripe Checkout session:

```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

// Create a checkout session
const session = await stripe.checkout.sessions.create({
  mode: 'subscription',
  payment_method_types: ['card'],
  line_items: [{ price: PRICE_ID, quantity: 1 }],
  success_url: 'https://app.blackroad.io/welcome?session_id={CHECKOUT_SESSION_ID}',
  cancel_url: 'https://app.blackroad.io/pricing',
  metadata: { blackroad_user_id: userId },
});

// Redirect customer
return { url: session.url };
```

---

## Webhooks

The Stripe webhook handler listens for subscription lifecycle events and provisions or revokes Converter API access accordingly.

### Registered webhook events (`stripe-config.json`)

| Event | Action |
|-------|--------|
| `checkout.session.completed` | Create subscription record, provision API token |
| `customer.subscription.created` | Activate Converter API access |
| `customer.subscription.updated` | Update rate limits based on new plan |
| `invoice.paid` | Renew token and confirm access |
| `customer.subscription.deleted` | Revoke Converter API token |

### Webhook handler

```typescript
import Stripe from 'stripe';
import { provisionToken, revokeToken } from './converter-api';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function handleStripeWebhook(
  rawBody: Buffer,
  signature: string
): Promise<void> {
  const event = stripe.webhooks.constructEvent(
    rawBody,
    signature,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  switch (event.type) {
    case 'checkout.session.completed':
    case 'customer.subscription.created':
    case 'invoice.paid':
      await provisionToken(event.data.object as Stripe.Subscription);
      break;

    case 'customer.subscription.updated':
      await updateTokenScope(event.data.object as Stripe.Subscription);
      break;

    case 'customer.subscription.deleted':
      await revokeToken(event.data.object as Stripe.Subscription);
      break;
  }
}
```

### Testing webhooks locally

```bash
# Forward Stripe events to your local server
stripe listen --forward-to localhost:8080/webhooks/stripe

# Trigger a test event
stripe trigger checkout.session.completed
```

---

## Subscription Status Check

The Converter API validates subscription status on every request:

```bash
GET https://converter.blackroad.io/subscription/status
Authorization: Bearer br_user_xxxx
```

```json
{
  "active": true,
  "plan": "pro",
  "requests_today": 1432,
  "limit_per_day": 10000,
  "renews_at": "2026-04-03T00:00:00Z"
}
```

---

## Customer Portal

Allow customers to manage their subscription (upgrade, downgrade, cancel):

```typescript
const portalSession = await stripe.billingPortal.sessions.create({
  customer: stripeCustomerId,
  return_url: 'https://app.blackroad.io/settings',
});

return { url: portalSession.url };
```

---

## References

- [stripe-config.json](../../stripe-config.json) — Product and webhook definitions
- [Vendor API Gateway](./vendor-api-gateway.md) — How subscriptions gate Converter API access
- [OAuth & Authentication](./oauth.md) — Token issuance tied to subscription status
