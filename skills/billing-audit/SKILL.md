---
name: billing-audit
description: Audits an app's payments and subscriptions for billing edge cases and reports everything it gets wrong. Use when asked to check, review, or audit billing, subscriptions, checkout, payment webhooks, or paid access, with Stripe, Dodo, Paddle, Lemon Squeezy, Polar, Razorpay, or any other provider, including marketplaces where admins or creators set the plans and members pay.
---

# Billing audit

Find what the app's billing gets wrong before customers do. Billing code that compiles proves nothing. Most real failures come from provider config and from cases nobody handled.

## How to audit

1. Find the billing code: checkout, provider calls, webhook handlers, subscription and access models, scheduled jobs, schema. Note the provider and its API version, then read that provider's current docs. Don't trust memory.
2. Work out what the app supports, like recurring or one-time plans, trials, plan changes, refunds, multiple currencies, or admin-created plans. Skip cases for features it doesn't have.
3. Go through every case under "What correct looks like" and trace the real code path for it. Don't judge from function names, comments, or docs.
4. In a large codebase, split the work by area, like setup and checkout, webhooks, and access, and audit the areas in parallel.
5. Read only. Don't change code unless the user asks.

## Report

Lead with the problems, worst first. Rank by harm: charging people wrongly, then wrong access, then lost revenue, then the rest.

For each problem give:

- What's wrong, in one sentence.
- A concrete scenario with the starting state, what happens, and the wrong result.
- File and line.
- The fix, in one line.
- How sure you are, and whether you traced it fully or are inferring.

Then three short lists:

- **Can't tell from code.** Dashboard settings and business decisions to ask the user about.
- **Handled well.** One line each, so the user knows what you checked.
- **Doesn't apply.** Cases you skipped and why.

## What correct looks like

### Provider setup

Some of this lives in the provider dashboard. Check it through the provider's API or CLI if you can. Otherwise list it under "Can't tell from code".

- The product is recurring, not one-time.
- Price and billing interval are right.
- The subscription keeps renewing. If the provider needs a total length or cycle count, it's set long, and something handles it running out. Check the provider docs for how this works.
- Trial length is what the business wants. Zero if none.
- Test and live mode each have their own products, keys, webhook URL, and webhook secret. Test products usually don't copy to live.
- The webhook endpoint subscribes to every event the code handles.
- Someone owns tax, the app or a merchant of record. Every charge produces a receipt or invoice.
- Money is stored as integers in each currency's smallest unit. Not every currency has 100 cents.

### Webhooks and API calls

- Every webhook is verified with the provider's mechanism, usually a signature over the raw body. No JSON parser or middleware touches the body first.
- The same event can arrive many times. Events are deduped by ID, and each effect is idempotent too. One paid invoice extends access once and sends one receipt.
- Events arrive out of order. Each event triggers a fetch of the subscription from the provider, or events older than the stored state get skipped. One subscription's events are processed one at a time. An old failed payment never overrides a newer successful one.
- The user is found through a provider ID stored at checkout, not webhook metadata. An event that arrives before its subscription or user exists in the DB gets retried later, not dropped.
- Every subscription status the provider can send is handled, not just active and canceled. Unknown statuses get logged and fail safe.
- If the provider says a payment is still processing, the event gets retried later, not marked done.
- The handler returns 2xx fast. A handler that dies halfway is safe to retry.
- Every call that creates a charge, subscription, or refund sends an idempotency key tied to the user's action. Double clicks and two open tabs can't start two checkouts. After a timeout, the code checks whether the call went through before trying again.
- The webhook can arrive late. The checkout success page asks the provider or shows a pending state.
- Login and the rest of the app keep working when the provider is down or sends a plan the code doesn't know. If billing records can't be read, automatic charges and suspensions pause instead of guessing.

### Access

- Access is computed in one place, from what the user actually paid for and until when. Not an `isPro` flag that events flip. An `active` status or a renewal event alone doesn't prove the money arrived.
- Period ends come from the provider's timestamps, stored in UTC. Access ends exactly at the boundary.
- DB, provider, and UI show the same plan and status.

### Cases to handle

- **Renewal payment fails.** There's a defined grace period or revoke. Retries are capped and stop on hard declines. Banks penalize endless retries.
- **Renewal needs the customer to authenticate.** They get a link to finish paying.
- **User updates their card.** Renewals and retries use the new card.
- **User cancels.** Access lasts until the period ends.
- **Provider cancels, pauses, or halts the subscription, or the user revokes autopay.** Access lasts for the paid period, renewals stop, and the user learns how to restart.
- **User comes back after a lapse.** Old unpaid invoices get collected, kept, or waived on purpose.
- **Refund or chargeback.** Access is revoked now or at period end, by a stated rule.
- **Upgrade or downgrade mid-cycle.** Proration and timing are defined. The user sees the real charge first. Unpaid time never gets credited. A failed upgrade payment leaves the old plan untouched.
- **User changes plans.** The existing subscription gets updated where the provider supports it. If it must be replaced, the old one is cancelled before the new one charges. Two live subscriptions never exist.
- **User buys a second plan while one is active.** It's blocked, converted to a plan change, or stacked on purpose, across every purchase path, one-time and recurring.
- **User starts checkout and never finishes.** The pending subscription gets cleaned up and doesn't block the next try.
- **User leaves, gets removed, or deletes their account.** The subscription is cancelled at the provider. Where that's impossible, as with app stores, the user is told how.
- **Trial ends with no payment method.** There's a defined outcome.

### What the customer sees

Auto-renewal and mandate rules differ by country and state. Ask the user which apply.

- Before the first charge, the user sees the price, how often it renews, and how to cancel, and agrees. That consent is stored.
- Users can cancel and update their card online, as easily as they signed up. Billing portal links are generated fresh each time.
- Renewal reminders, pre-debit notices, and price-change notices go out where the rules require them.

### When admins create the plans

Applies when a community admin, creator, or seller creates the tier and members pay.

- **Admin edits price or term.** Existing subscribers keep their old terms unless moved on purpose.
- **Admin archives or deletes a tier.** Existing subscribers keep renewing, get moved, or get cancelled, by a stated rule.
- **Admin's payout account gets suspended, paid plans get turned off, or the community gets suspended or deleted.** Renewals stop, or collection continues with the money held on purpose. Nothing keeps charging for access that no longer exists.
- **Payout to the admin fails.** It gets retried or flagged, never dropped.

### Monitoring

- Every webhook is logged with event ID, type, subscription ID, and result.
- A scheduled job compares subscriptions, invoices, and charges with the provider and fixes or flags drift. A renewal whose webhook never arrived still counts.
- An alert fires when webhooks stop arriving. Providers disable endpoints that keep failing.
