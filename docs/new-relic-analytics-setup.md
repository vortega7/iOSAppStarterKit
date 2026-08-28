# New Relic analytics setup

A specific, reasoned New Relic mobile agent configuration, a baseline
custom-event taxonomy worth reusing, and a real gap worth fixing from
day one rather than discovering it later.

## SDK setup

Add the New Relic iOS agent SPM package
(`https://github.com/newrelic/newrelic-ios-agent-spm`). Configure it as
early as possible in the app lifecycle — `NewRelic.start` needs to run
close to process launch, on the main thread, which is why this needs an
actual `AppDelegate` bridged in via `@UIApplicationDelegateAdaptor`
even in an otherwise-pure-SwiftUI app; there's no equivalent SwiftUI-
native hook that fires early enough.

## Feature flags — set these deliberately, not just defaults

A specific configuration, with the reasoning per flag. Re-evaluate each
one for the new app's actual traffic shape/data-volume budget rather
than copying blind, but treat "did I consider each of these" as the
actual requirement, not "did I copy the defaults":

```swift
// Required, not optional, if the app uses Swift's async/await
// URLSession API (very likely, on a modern SwiftUI app) — the agent
// does not instrument that API shape unless this is explicitly
// enabled. Without it, no HTTP request/response data is captured at
// all, despite that typically being the primary reason for the
// integration.
NewRelic.enableFeatures(NRMAFeatureFlags.NRFeatureFlag_SwiftAsyncURLSessionSupport)

// If there's no New Relic-instrumented backend to correlate traces
// with (the app calls third-party APIs directly, no backend this app
// controls), distributed tracing headers/spans are pure overhead.
NewRelic.disableFeatures(NRMAFeatureFlags.NRFeatureFlag_DistributedTracing)

// The single biggest data-volume risk for many AI-API-backed apps:
// request/response bodies can include large payloads (base64 audio,
// images, video frames). Capturing response bodies burns data budget
// fast for little value — timing and status are usually what's
// actually needed.
NewRelic.disableFeatures(NRMAFeatureFlags.NRFeatureFlag_HttpResponseBodyCapture)

// Auto UI-interaction tracing is built around UIKit view
// controller push/pop and has little value for a SwiftUI app,
// especially a single- or few-screen one.
NewRelic.disableFeatures(NRMAFeatureFlags.NRFeatureFlag_InteractionTracing)
NewRelic.disableFeatures(NRMAFeatureFlags.NRFeatureFlag_DefaultInteractions)

// Skip if there are no WebViews anywhere in the app.
NewRelic.disableFeatures(NRMAFeatureFlags.NRFeatureFlag_WebViewInstrumentation)

// Leave AutoCollectLogs at its default (disabled) unless there's a
// specific reason to enable it — it ships every print/stdout line as
// remote log data, an easy volume trap to miss.

// If this app has any usage-metered feature, set a stable per-device/
// per-user ID here, before `start` — see "Usage metering" below for
// why, and for the read-or-create-against-a-shared-Keychain-account
// pattern if the ID needs to be readable from more than one call site.
// let deviceID = /* read-or-create from Keychain */
// NewRelic.setUserId(deviceID)

NewRelic.start(withApplicationToken: "[FILL IN: app token from the New Relic dashboard]")
```

New Relic has a free-tier data budget (check current terms — historically
around 100GB/month) — every flag above is either required for traffic
to be visible at all, or an explicit opt-out of something that would
burn that budget for no value this specific app gets. Revisit the
budget/flags if the new app's traffic shape is materially different
(e.g. it does stream large media payloads and actually needs response
capture for debugging).

## `AnalyticsRecording` protocol

One minimal Domain protocol (`recordEvent(_:attributes:)`, plus
`recordHandledError(_:attributes:)` if the app wants handled-error
reporting), one concrete `NewRelicAnalyticsService` implementation.
Every store/ViewModel that needs to log an event takes this as an
injected dependency — never calls `NewRelic.recordCustomEvent` directly
— so it's fakeable in tests and swappable if the analytics vendor ever
changes.

**Gotcha worth knowing before reaching for the SDK method by name**:
for a native Swift `Error`, the correct New Relic call is
`NewRelic.recordError(_ error: NSError, attributes:)` (it bridges a
Swift `Error` to `NSError` automatically) — **not**
`recordHandledException`, which expects an Objective-C `NSException`
and is easy to reach for instead purely because the name reads closer
to a Domain method called `recordHandledError`.

## Baseline custom-event taxonomy

One event **name** per feature area, with an `outcome` (or similar)
**attribute** distinguishing cases — not one event name per outcome.
This shape makes New Relic dashboards/alerts filter by attribute rather
than needing a union of many event names:

| Event name | `outcome` values (example) | Fires from |
|---|---|---|
| `[FILL IN, e.g. SubscriptionLifecycle]` | `offered` / `signedUp` / `declined` / `renewed` / `notRenewed` / `pending` / `purchaseFailed` / `restoreFailed` / `productLoadFailed` / `trialReplayBlocked` | `SubscriptionStore` — **only** from `observeTransactionEvents()`'s stream for signedUp/renewed/notRenewed, never also from the `purchase()` return value, or a purchase this app instance drives gets double-counted (see `LESSONS_LEARNED.md` #6 for the related transaction-ID de-dup issue) |
| `RemoteConfigSync` | `success: Bool` + one changed-flag per synced parameter | `SettingsStore.syncFromRemoteConfig()`, every fetch attempt |
| `LegalAcceptance` | `termsAccepted` / `[any other consent]Accepted`, plus `acceptedVia: "onboarding"` vs `"reacceptance"` | `LegalAcceptanceStore`, whenever a gate is accepted |
| `[FILL IN: usage-metering event, e.g. TranslationUsage/GenerationUsage]` | `secondsRecorded`/`unitsConsumed` (numeric) + `isSubscribed` (Bool) | The store owning entitlement, flushed on backgrounding — see "Usage metering" below |
| `[FILL IN: the app's core feature error/status event, if applicable]` | | |

**Known gap worth fixing from day one, not retrofitting later**: if
this app has any usage-metered feature (a per-session, per-minute,
per-generation cost), send the actual usage quantity as a numeric event
attribute on every completed unit of usage, and set a **stable per-user
identifier** on New Relic events (`NewRelic.setUserId(_:)` or
equivalent — e.g. a StoreKit original-transaction-ID for subscribers, a
persisted anonymous UUID for trial users). Without both of these, there
is no way to build a "usage per user this month" dashboard later — New
Relic's own automatic per-install device ID isn't enough to reliably
tie usage back to a specific paying customer, and per-turn usage that's
only ever accumulated into a local on-device counter (not telemetered)
can't be aggregated or reported on at all. It's common to ship a v1
without either of these and have to add them as a follow-up — build
both in from the start instead.

### Usage metering: batch on backgrounding, don't send one event per unit of usage

Built and confirmed end-to-end in a real app (TranslationApp, 2026-08):
sending one custom event per completed unit of usage (per recording, per
generation, per API call) creates an unbounded, ever-growing event table
for zero benefit — every question worth answering ("seconds per user
this month," "total usage this week") is a `SUM`/`FACET` over the raw
numeric attribute regardless of how many rows it's spread across.

**Pattern**: accumulate usage into an in-memory (not persisted) buffer
on the store that already owns entitlement/usage decisions — don't
persist the buffer, and don't add a new store just for this. Flush the
buffer as one event, with the accumulated total as the numeric
attribute, only on an app-backgrounding transition (SwiftUI
`scenePhase == .background`) — not per-unit-of-usage, and not on a
periodic timer (keeps the reporting boundary singular and simple).
Reset the buffer to zero after each flush, so the next flush reports
only what's new, not a repeating cumulative total.

**Accepted tradeoff, state it explicitly wherever this is documented**:
a force-quit before the app is ever backgrounded loses that session's
seconds from New Relic's view. This does **not** need to affect
enforcement accuracy — keep the real, authoritative usage counter (the
one that actually gates behavior) `UserDefaults`-persisted and written
immediately on every unit of usage, entirely independent of the
in-memory reporting buffer. New Relic's copy of the number is allowed to
be eventually-consistent; the enforcement copy isn't.

**Per-device identifier without real per-user identity yet**: if the
app doesn't yet tie usage to a real account (no backend, subscriptions
are device-local StoreKit entitlements), attribute usage to a
self-generated, Keychain-persisted UUID rather than New Relic's own
automatic per-install device ID — Keychain survives reinstall, an
automatic device ID may not, and a Keychain UUID reads as a real stable
identifier once accounts exist later. Set it once via
`NewRelic.setUserId(_:)` at launch, before `NewRelic.start(...)`. If
launch-sequencing between the app delegate and wherever the entitlement
store gets constructed isn't guaranteed, have both call sites
independently read-or-create the value against the same Keychain
account — idempotent regardless of which runs first, and avoids needing
to depend on that ordering at all.

## Dashboard-building notes

NRQL (New Relic's query language) can compute time-windowed aggregates
directly from raw per-event data — e.g. "sum of usage seconds this
calendar month, grouped by user ID" — without needing to build any
monthly-reset logic client-side, as long as the raw events carry both
the numeric usage value and the stable user identifier above. Design the
event schema with this in mind before writing the first line of
dashboard/NRQL, not after.
