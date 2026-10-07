---
name: billing-audit
description: Audit billing and subscriptions for edge-case bugs. Use when the user wants their billing, subscriptions, or payment webhooks audited.
---

# Billing Audit

Billing code that compiles proves nothing. Real failures come from provider config and from cases nobody handled. This skill **traces** every case in the Reference through the real code and reports where the app gets it wrong.

## Redact

Billing code sits next to live keys and customer data. Write `<REDACTED>` in place of every API key, webhook secret, and customer email, name, or card detail you quote. Calls to the provider are read-only: fetch and list, nothing that creates, updates, or charges.

## Process

### 1. Map the money flow

Read the README, product docs, domain glossary, pricing page, and schema. Write down the **money flow**:

- What's sold, and who pays whom: the business directly, or sellers and creators through the platform.
- The provider, its API version, and the billing features in use: recurring, one-time, trials, seats, usage, plan changes, refunds, currencies.
- Where customers are. Renewal and mandate rules differ by region.
- **Documented decisions**: billing behavior the docs choose on purpose, like "no refunds". The code is judged against these.

Done when you can state the money flow in three lines. It opens the report.

### 2. Fit the cases to the project

Mark each case in the Reference that the project's features rule out as **doesn't apply**. Add the cases the money flow implies and the Reference lacks. If sellers set prices, cover a seller changing a price, leaving, or getting suspended.

### 3. Find the billing code and the provider docs

Locate checkout, provider calls, webhook handlers, subscription and access models, scheduled jobs, and schema. Read the provider's current docs for the API version in use. Provider behavior shifts between versions, and the docs are the source of truth for it.

### 4. Trace every case

For each case, **trace** the real code path from trigger to stored state to what the user sees. Docs give intent; the traced path gives behavior. A gap between them is a problem.

Every case lands in exactly one bucket:

- **Problem**: the traced path produces a wrong charge, wrong access, lost revenue, or other harm.
- **Handled well**: traced, and it matches the Reference.
- **Can't tell from code**: the answer lives in a dashboard setting or a business decision.
- **Doesn't apply**: ruled out in step 2.

Done when every case has a bucket.

In a large codebase, split by area (setup and checkout, webhooks, access and cases) and run one sub-agent per area in parallel. Each sub-agent prompt includes:

- The money flow and documented decisions from step 1.
- Its sections of the Reference, pasted in full. The sub-agent has no other access to them.
- The Redact rules and the four buckets.
- The brief: "Trace every case in your sections and put each in exactly one bucket. For each problem, give the scenario, file and line, fix, and confidence."

### 5. Report

Present, in order:

1. The money flow in three lines, so the user can catch a wrong assumption early.
2. **Problems**, ranked by harm: wrong charges, then wrong access, then lost revenue, then the rest. For each:
   - What's wrong, in one sentence.
   - The scenario: starting state, what happens, wrong result.
   - File and line.
   - The fix, in one line.
   - Confidence: fully traced, or inferred.
3. **Can't tell from code**, as questions for the user.
4. **Handled well**, one line each.
5. **Doesn't apply**, each with its reason.

Stop after the report. Fixes start when the user asks for them.

## Reference

What correct billing looks like. Each bullet is one case.

### Provider setup

Some of these live in the provider dashboard. Read them through the provider's API or CLI where you can; the rest go to **Can't tell from code**.

- The product is recurring.
- Price and billing interval match the pricing page.
- The subscription renews indefinitely. Where the provider requires a total length or cycle count, it's set long and something handles reaching it.
- Trial length matches what the business wants, zero if none.
- Test and live mode each have their own products, keys, webhook URL, and webhook secret. Products made in test mode usually need recreating in live.
- The webhook endpoint subscribes to every event the code handles.
- Tax has an owner, the app or a merchant of record, and every charge produces a receipt or invoice.
- Money is stored as integers in each currency's smallest unit, with that currency's own precision. Some currencies, like the yen, have no minor unit.

### Webhooks and API calls

- Every webhook is verified with the provider's mechanism, usually a signature computed over the raw body before any parser reads it.
- Delivery is at-least-once. Events are deduped by ID, and each effect is idempotent: one paid invoice extends access once and sends one receipt.
- Delivery order is arbitrary. Each event triggers a fresh fetch of the subscription from the provider, or events older than the stored state get skipped. One subscription's events are processed one at a time, so the newest payment outcome always wins.
- The user is looked up by a provider ID stored at checkout. An event that arrives before its subscription or user exists gets retried until they do.
- Every subscription status the provider can send has a handler. Unknown statuses get logged and fail safe.
- An event whose payment is still processing gets retried and stays open until the outcome is known.
- The handler acknowledges fast and is safe to rerun after dying halfway.
- Every call that creates a charge, subscription, or refund carries an idempotency key tied to the user's action, so double clicks and two open tabs start one checkout. After a timeout, the code checks whether the call went through before retrying.
- The checkout success page asks the provider for the result or shows a pending state, since the webhook can lag.
- Login and the rest of the app keep working when the provider is down or sends an unknown plan. When billing records are unreadable, automatic charges and suspensions pause until they're readable again.

### Access

- Access is derived in one place from paid periods: what the user paid for, and until when. An `active` status or a renewal event counts once the payment is confirmed.
- Period ends come from the provider's timestamps, stored in UTC, and access ends exactly at the boundary.
- The DB, the provider, and the UI show the same plan and status. Any gap between them is **drift**.

### Cases

- **Renewal payment fails.** A defined grace period or immediate revoke applies. Retries are capped and stop on hard declines, since banks penalize repeated retries.
- **Renewal needs the customer to authenticate.** They get a link to finish paying.
- **User updates their card.** Renewals and retries charge the new card.
- **User cancels.** Access lasts until the paid period ends.
- **Provider cancels, pauses, or halts the subscription, or the user revokes autopay.** Access lasts for the paid period, renewals stop, and the user learns how to restart.
- **User returns after a lapse.** Old unpaid invoices get collected, kept, or waived by a stated rule.
- **Refund or chargeback.** Access ends now or at period end, by a stated rule.
- **Upgrade or downgrade mid-cycle.** Proration and timing are defined, and the user sees the real charge before confirming. Credits cover only paid time. A failed upgrade payment leaves the old plan exactly as it was.
- **User changes plans.** The existing subscription gets updated where the provider supports it. Where it must be replaced, the old one is cancelled before the new one charges, so the user holds one live subscription at a time.
- **User buys a second plan while one is active.** It's blocked, converted to a plan change, or stacked on purpose, the same way across every purchase path, one-time and recurring.
- **User abandons checkout.** The pending subscription gets cleaned up, and the next attempt starts fresh.
- **User leaves, gets removed, or deletes their account.** The subscription is cancelled at the provider. Where the app can't cancel it, as with app stores, the user gets instructions.
- **Trial ends with no payment method.** A defined outcome applies.
- **A plan's price changes.** Existing subscribers keep their old price until moved on purpose.
- **A plan gets retired.** Existing subscribers keep renewing, get moved, or get cancelled, by a stated rule.

### What the customer sees

Auto-renewal and mandate rules differ by country and state. Ask the user which apply.

- Before the first charge, the user sees the price, renewal interval, and how to cancel, and agrees. The consent is stored.
- Cancelling and updating a card work online, as easily as signing up. Billing portal links are generated fresh each time.
- Renewal reminders, pre-debit notices, and price-change notices go out where the rules require them.

### Monitoring

- Every webhook is logged with event ID, type, subscription ID, and result.
- A scheduled job compares subscriptions, invoices, and charges with the provider and fixes or flags **drift**, including renewals whose webhook never arrived.
- An alert fires when webhooks stop arriving, since providers disable endpoints that keep failing.
