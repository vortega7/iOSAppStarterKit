# Subscription / free-trial paywall pattern

A reusable design for a free-tier + StoreKit 2 subscription gate,
including three status-read false starts worth skipping straight past.
This is typically the single most complex piece of "baseline
infrastructure" in an app like this — budget real time for it, and
expect real on-device (not just Simulator) testing before trusting any
of it.

## Product decisions to make explicitly for the new app (don't default blindly)

Each of these is a real product decision, not a technical default —
confirm each one explicitly for the new app rather than assuming the
example choice below is universal:

- **No-payment-info free trial, or pay-upfront-with-a-short-trial?** A
  true no-payment-info trial (no credit card collected at all) is a
  real, defensible product choice — some users specifically avoid any
  trial that requires upfront payment info, distrusting the auto-renew
  mechanic. It comes at a cost: it can't rely on Apple's own per-Apple-
  ID introductory-offer eligibility tracking, which is what stops
  reinstall abuse for apps that *do* collect payment info upfront — so
  a no-payment-info trial needs its own reinstall-abuse mitigation (see
  below). Decide this tradeoff deliberately, not by default.
- **Usage metric**: define exactly what counts toward the free
  allowance — e.g. only active input-capture time (microphone
  recording, camera-active time), specifically *not*
  processing/network-wait time or output-playback time. [FILL IN:
  decide this app's equivalent — e.g. generation requests, seconds of
  video output, etc.]
- **Mid-session cutoff behavior**: when the free allowance is crossed
  *during* an active session (not just checked at session-start), stop
  the in-flight session and discard its output — never send a paid API
  call for usage that's already past the free allowance — and show the
  paywall immediately. The alternative (let the in-flight session
  finish, only gate the *next* one) is a softer UX but lets usage
  overshoot the limit by up to one full session; decide which tradeoff
  fits the product.
- **Reinstall-abuse tolerance**: decide how strict this needs to be. A
  deliberately *loose* deterrent (not a hard ban) is a defensible
  choice if the goal is only to stop a rapid delete/reinstall/repeat
  loop, not to catch every possible edge case — in that case, don't
  over-engineer this into a device-fingerprinting arms race; a
  Keychain-stored timestamp plus a generous cooldown window can be the
  entire mechanism.

## App Store Review requirements for the purchase flow — not optional, build these in from day one

Two real App Store rejections' worth of requirements (Guidelines
3.1.2(c) and 2.1(b)), confirmed the hard way on a shipped app. Both are
easy to miss because the app *works* without them — they only surface
once a real reviewer tries to test the purchase:

- **The subscription's title, length, price, and functional links to
  the Privacy Policy *and* Terms of Use must be visible directly in the
  purchase flow screen itself** — not just reachable elsewhere in the
  app (a Settings "Legal" section with the same links does **not**
  satisfy this; the paywall view needs its own links too). If the app
  uses Apple's Standard EULA (no Custom EULA configured in App Store
  Connect), the App Store **Description** also needs a real, clickable
  Terms of Use URL — plain-text prose like "see our Terms" with no
  actual URL does not count as a functional link.
- **The purchase flow must be reachable without needing to exhaust any
  usage limit first.** A paywall that only appears reactively (once a
  free allowance runs out) is not something App Review can practically
  test — add a direct entry point (e.g. a "Subscribe" row in Settings,
  visible whenever the user isn't already subscribed, that opens the
  paywall directly) from the first version that ships a paywall, not as
  a fix after a rejection.

Both of these are pure UI additions — they don't change any entitlement/
limit-checking logic, just where the purchase screen is reachable from
and what it displays. Cheap to build in from the start; expensive to
retrofit after a rejection costs a review cycle.

## Architecture

- `SubscriptionManaging` (Domain protocol) hides StoreKit 2
  (`Product`, `Transaction`, `Transaction.updates`) behind Domain-native
  types (`SubscriptionProduct`/`SubscriptionStatus`/`SubscriptionOutcome`/
  `SubscriptionTransactionEvent`) — same pattern as any other Data-layer
  boundary in this app family.
- `SubscriptionStore` (`@Observable`) owns: cached subscription status,
  cumulative free-usage-consumed counter, paywall presentation state,
  every analytics event for the subscription lifecycle (see
  `docs/new-relic-analytics-setup.md`'s event taxonomy), and — if the app
  has post-trial fair-use metering — the monthly usage counter/throttle
  tier (see "Fair-use throttling for post-trial/subscribed usage"
  below). Takes a `SettingsStore` dependency to read the Remote-Config-
  sourced free-usage limit (and the fair-use thresholds/delays, if
  applicable), rather than running an independent Remote Config fetch
  cycle.
- `isOverLimit(afterAdding:)` is the **single source of truth** for the
  limit rule — every gate (start-of-session check, live mid-session
  check) goes through it, never duplicates the arithmetic.

## StoreKit 2 status reads: use `Transaction.all`, not the higher-level APIs

This needed **three** different implementations before one actually held
up under real on-device purchase/refund/relaunch testing — see
`LESSONS_LEARNED.md` #5 for the full story. Start directly with the
third approach on a new app rather than re-discovering the first two
don't work:

```swift
// Read every transaction directly, compute entitlement from a
// matching transaction's own revocationDate/expirationDate — more
// reliable on-device than Transaction.currentEntitlements or
// Product.SubscriptionInfo.status, both of which were independently
// found to sometimes report no entitlement for a transaction Xcode's
// own Transaction Manager confirmed was active.
for await result in Transaction.all {
    // ...
}
```

## Three independent triggers keep cached status fresh

`SubscriptionStore.status` is a cached value (gating needs a synchronous
answer), refreshed by three independent triggers, each catching a case
the others miss:
1. **Foreground resume** — cold launch + every time the app returns to
   foreground.
2. **`Transaction.updates` stream** — consumed for the app's entire
   lifetime, catches purchases/renewals/revocations that happen without
   this app instance driving them. **De-duplicate by transaction ID,
   persisted across launches** — this stream replays current/recent
   transactions on every fresh subscription to it, in practice on every
   launch, not just genuinely new events (see `LESSONS_LEARNED.md` #6).
3. **A periodic timer** (e.g. every 5 minutes) — catches a user who
   keeps the app open and foregrounded continuously (no background/
   foreground transition ever fires, and StoreKit's transaction push
   isn't reliably prompt for a local-testing revocation).

**Purchase-confirmation race**: StoreKit's own purchase sheet toggles
scene phase as it presents/dismisses, which can trigger a foreground-
resume status refresh *before* the underlying StoreKit read has caught
up with a transaction verified moments earlier. A short grace period
(remember "we just confirmed a purchase, don't trust a stale
not-subscribed read for the next N seconds") is needed, or a
just-completed purchase can appear to revert.

## Local StoreKit Testing — practical notes

- The local `.storekit` configuration only attaches when **Xcode itself
  launches the app** for the scheme it's wired to (Run/Cmd+R). A plain
  `simctl launch`, or the app reopened from the Home screen after being
  backgrounded, won't have it — StoreKit calls will instead try to reach
  the real App Store and fail to find any product.
- Don't edit the `.storekit` file's settings (e.g. subscription renewal
  rate) while the app is actively running against it — this forces
  StoreKit Testing's local environment to reload, disconnecting the
  running app's view of its own transactions. Stop the app, change the
  setting, then Run again.
- **Known, unfixable-from-the-app limitation**: restoring a purchase
  after deleting and reinstalling the app does not work against a
  *local* StoreKit Testing configuration, no matter which status-read
  API is used — see `LESSONS_LEARNED.md` #5. Don't re-chase this
  scenario locally; genuinely validating restore needs a real (or Apple
  Sandbox) Apple ID against a real App Store Connect product.

## Failure states that need an explicit UI, not just a spinner

`try?` around any `SubscriptionManaging` call is a recurring footgun
(see `LESSONS_LEARNED.md` #8) — a silently-discarded product-load
failure can leave the paywall spinning forever with no Subscribe button
and no retry. Every StoreKit call in the paywall path needs an explicit
failure state with a retry affordance, not just a loading/success
binary.

## Testing

- `SubscriptionStoreTests`: usage/limit math against a fake
  `SubscriptionManaging` and fake `SettingsStore`/credential store (see
  `LESSONS_LEARNED.md` #11 for the test-isolation footgun to avoid), and
  the analytics outcome recorded for every purchase/restore/
  transaction-stream/status-refresh path via a test-controlled
  `AsyncStream` for transaction events.
- The concrete `StoreKitSubscriptionService` itself is deliberately
  *not* unit-tested — exercise it manually via the local `.storekit`
  configuration instead; StoreKit Testing's simulated purchase flows
  aren't something `xcodebuild test` can drive deterministically.

## Fair-use throttling for post-trial/subscribed usage

If the product has a disclosed fair-use clause in its Terms of Use
(e.g. "excessive use over N minutes/month will be limited") that
applies **regardless of subscription status** — a paying subscriber's
usage is still metered, not just the free trial's — this is a separate
mechanism from the free-trial gate above, built and confirmed
end-to-end in a real app (TranslationApp, 2026-08):

- **Enforce as escalating throttling, never a hard block.** A soft
  threshold adds a short artificial delay before the result is
  delivered/played; a second, higher "hard" threshold increases that
  delay further. A subscriber is never fully locked out — the paywall's
  entire value proposition is "you're paying, so you keep working,"
  and a hard block there would undermine that.
- **A separate counter from the lifetime free-trial counter**, since
  this one resets every calendar month rather than accumulating
  forever. Track the month it currently represents as a `"yyyy-MM"`
  string (year **and** month — a month-alone comparison incorrectly
  treats e.g. January this year and January next year as the same
  month) and roll the counter over whenever the current month no
  longer matches. Make the "what time is it" source an injectable
  closure on the owning store's initializer (defaulting to
  `Date.init`), so tests can simulate crossing a month boundary without
  waiting for a real one.
- **Both thresholds and both delay amounts should be Remote-Config-tunable**,
  not hardcoded — you won't know the right values until you have real
  usage data, and a hardcoded value means a full app-review cycle to
  adjust it later.
- **The delay must actually block starting a new unit of work while
  it's in effect**, not just add latency to the current one's output —
  otherwise a throttled user can keep queuing new requests at full
  speed and only feel the delay on results, never on their actual pace
  of usage, which defeats the purpose. If the app has a state machine
  gating new actions on an idle-like state, give the delay its own
  distinct state (not the same state as "delivering a normal result")
  so the delay window is both enforced correctly *and* rendered with
  accurate status text — reusing an existing "in progress" state for
  this works functionally but reads as misleading UI copy during the
  wait (a "Playing…"-style label showing before anything has actually
  started playing, discovered on real-device testing).
- **This counter resets on app reinstall** if it's only
  `UserDefaults`-backed (uninstall wipes `UserDefaults`), same as the
  free-trial counter's gap above — but treat this as a *lower*-priority
  gap than the free-trial one, deliberately, not an oversight: the
  free-trial gate protects against getting the entire product for free
  via repeated reinstalls (worth the Keychain-mirroring cost to fix);
  a fair-use throttle only adds a few seconds of friction for an
  already-paying subscriber, so a user motivated enough to reinstall
  repeatedly just to dodge that friction is a low-value threat to
  defend against. Decide this tradeoff explicitly for your product
  rather than copying either answer blind.
- Report actual usage to your analytics platform for this too — see
  `docs/new-relic-analytics-setup.md`'s "Usage metering: batch on
  backgrounding" section for the reporting-cadence pattern that pairs
  with this.

## Known, accepted v1 gap: this is all client-side enforcement

Every mechanism above is a UX layer, not real security enforcement — a
technically capable user can bypass any purely client-side gate,
especially since this app calls its core paid API directly from the
client with an embedded key (no backend proxy). See
`LESSONS_LEARNED.md` #12. Real enforcement needs a backend + server-to-
server purchase-notification webhook (App Store Server Notifications V2
on iOS) — scope that as a deliberate, separate decision, not something
to bolt onto this client-side MVP incrementally.
