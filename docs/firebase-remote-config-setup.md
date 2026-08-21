# Firebase Remote Config setup

What Firebase Remote Config is for in an app of this shape, the sync
pattern to use, and a baseline parameter set worth including from day
one.

## Why Remote Config, and what it's for here

Two categories of value, both centrally managed without an app release:
1. **Secrets/limits that shouldn't ship hardcoded or be user-editable**
   — e.g. a shared third-party API key, a usage limit.
2. **Operational levers** — a forced-update minimum version, a fair-use
   threshold — that need to be changeable in an emergency without
   waiting for App Store review.

## Project setup

1. Create a Firebase project (console.firebase.google.com) — reuse an
   existing one if this app shares infrastructure with another, or
   create a dedicated one.
2. Add an iOS app to the project, matching this app's exact bundle ID.
   Download `GoogleService-Info.plist`, add it to the Xcode project
   (not tracked in git if it contains anything sensitive — check
   Firebase's current guidance on this; for Remote Config alone it's
   typically safe to commit, since the values it unlocks are still
   gated by Firebase project permissions).
3. Enable **Remote Config** in the console. Add parameters (see below)
   with sensible defaults matching the app's fallback constants.
4. If also hosting legal pages (see `docs/legal-compliance-checklist.md`),
   enable **Firebase Hosting** on the same project — one project, two
   features, no reason to split them.
5. `[FILL IN]` — record this project's Firebase project ID somewhere
   durable (the new app's own README, or wherever project-specific
   config is tracked) once created.

## Baseline parameters, exact names, and the business logic behind each

Five parameters worth carrying forward as a baseline, all synced by one
`syncFromRemoteConfig()` call. Four of the five are generic enough to
reuse as-is on any subscription-gated, cloud-API-backed app — only the
first is inherently product-specific. Treat this as an actual spec, not
just a naming reference: it captures *why* each value exists and
exactly how it's enforced, which is the part that's easy to lose if you
only copy the parameter names.

### 1. `[FILL IN: this app's core API key parameter name]` (String)

The shared third-party API key the app authenticates its core cloud-API
calls with. Centrally managed via Remote Config specifically so it's
**never** a `SettingsView`-editable preference and never ships hardcoded
in the binary — it's rotatable without an app release, and one key
change propagates to every install on next sync.

**Business logic**: `SettingsStore` stores the last-successfully-fetched
value Keychain-backed, via `SecureCredentialStoring` — not
`UserDefaults`, since an API key is a credential — so an offline launch
still has *a* key to use, even if it's stale. `syncFromRemoteConfig()`
overwrites it only when the fetched value differs from what's already
stored (avoids a redundant Keychain write on every sync). If this app's
core feature can't function without a key at all (no key ever
successfully fetched, e.g. a brand-new offline install), the relevant
call site should fail with a clear "not ready yet" error rather than
silently sending an empty/invalid key to the third-party API.

### 2. `free_usage_seconds` (Number)

**The generic free-tier usage allowance, in seconds.** Rename the key
to match this app's actual unit of usage (e.g. `free_generation_seconds`,
`free_translation_seconds`, `free_credits` if usage isn't time-based at
all) but keep the underlying mechanism identical.

**Business logic** (see `docs/subscription-paywall-pattern.md` for the
full design, this is the summary):
- A **cumulative, lifetime** allowance, not a recurring monthly one —
  granted once per install, never resets. (A monthly-resetting
  allowance instead is a deliberate, different design — the
  reset-tracking logic doesn't exist in this pattern as-is and would
  need to be added.)
- Pick a pre-first-fetch fallback default (used until the first
  successful Remote Config fetch ever completes, or if a fetch response
  ever omits the parameter) — never let the gate be accidentally
  "unlimited" before Remote Config has been reachable even once.
- Enforced two ways, both going through one `isOverLimit(afterAdding:)`
  source of truth: a **start-of-turn gate** (refuse before starting if
  already over) and a **live mid-session cutoff** (a repeating check
  while usage is actively accruing, which stops an in-flight session
  and discards its output the moment the limit is crossed *mid-turn* —
  a single start-of-turn check can't catch that case).
- What counts as "usage" needs an explicit, deliberate definition per
  app — e.g. only active input-capture time (recording, camera, etc.),
  specifically *not* processing/network wait time or output-playback
  time. Decide and document this early; it directly determines both the
  cost-model math (see `LESSONS_LEARNED.md` #13) and what the live
  mid-session monitor actually measures.

### 3. `free_trial_cooldown_days` (Number)

Reinstall-abuse mitigation for a **true no-payment-info free trial**
(no credit card collected upfront — see `docs/subscription-paywall-pattern.md`
for why this product decision rules out relying on Apple's own
per-Apple-ID introductory-offer tracking, which is what stops reinstall
abuse for apps that *do* collect payment info upfront).

**Business logic**: a deliberately loose default (e.g. `10` days). The
intent is only to stop a rapid delete/reinstall/repeat loop, not to
gate an occasional genuine returning user. A device that reinstalls
within this many days of its last trial grant gets silently blocked
from a fresh allowance (see `LESSONS_LEARNED.md` #9 for the non-obvious
Keychain-mirroring this requires, since the check runs synchronously
before this launch's own Remote Config fetch can complete, and
`UserDefaults` alone doesn't survive the reinstall being detected). The
blocked state should show the *same* generic "time's up" paywall
message as any other exhausted-limit case — not a distinct "we detected
you're reinstalling" message, so the mechanism isn't revealed to
someone probing for a bypass.

### 4. `minimum_app_version` (String)

The forced-update kill switch floor — see `ARCHITECTURE.md`'s "Forced-
update kill switch" for the full mechanism (root-level `WindowGroup`
gate, `AppVersion` numeric comparison, no bypass).

**Business logic**: default `"0.0.0"` — maximally permissive, used
until the first successful fetch ever completes. This specific default
matters: it means a first-ever launch that happens to be offline is
never incorrectly blocked before it's had any chance to fetch a real
value. Compared against the running build's own
`CFBundleShortVersionString` using real `major.minor.patch` numeric
comparison (not `String <`, which sorts `"1.10"` before `"1.9"`
lexically) — see `Domain/Models/AppVersion.swift`'s equivalent in the
new app.

### 5. `app_store_url` (String)

Paired with #4 — where the kill switch's "Update Now" button sends a
blocked user. No numeric default needed (an empty string just makes the
button a no-op until this is set); set it once the app has a real App
Store listing URL, and it can stay wrong/placeholder harmlessly until
`minimum_app_version` is ever actually raised above the current build
for the first time.

### Suggested generic parameter table for a new app

| Parameter key | Type | Default until first fetch |
|---|---|---|
| `[FILL IN: this app's core API key parameter name]` | String | — (see #1 above) |
| `free_usage_seconds` (or this app's equivalent unit) | Number | `[FILL IN]` |
| `free_trial_cooldown_days` | Number | `10` |
| `minimum_app_version` | String | `"0.0.0"` |
| `app_store_url` | String | `""` |

## The sync pattern

One `RemoteConfiguring` protocol, one `SettingsStore.syncFromRemoteConfig()`
method that fetches everything above in a single call (not one
independent fetch per store/feature — Remote Config is a single
process-wide config object; a second concurrent consumer fetching the
same thing on the same triggers is pure duplication).

**Fetch triggers**: cold launch, and every foreground resume (`.onChange
of scenePhase` or equivalent) — not a continuous polling timer. This is
a deliberate tradeoff, worth confirming explicitly rather than assuming:
a limit/config change while the app stays continuously foregrounded can
take up to Remote Config's own throttle window to apply, in exchange
for not adding a polling timer's overhead. Decide this explicitly for
each new app rather than defaulting blindly — it may not be the right
tradeoff for every product.

**Failure handling**: a failed fetch (offline, throttled) should always
mean "keep using whatever values are already stored" — never a
user-visible error. The whole point of Remote Config as an operational
lever is that it degrades gracefully to the last-known-good value, not
that it becomes a new source of crashes/errors when unreachable.

**Analytics**: log a single custom event per sync attempt (e.g.
`RemoteConfigSync`) with a `success` boolean and one changed-flag
attribute per parameter that actually changed — more directly queryable
than inferring "did anything change" from HTTP status alone, and lets
you build a dashboard answering "how often does this actually change in
production."

## The one genuinely subtle gotcha: synchronous, pre-sync reads

If any value needs to be read **synchronously**, very early in app
launch — before this launch's own async Remote Config fetch can
possibly have completed — a plain `UserDefaults`-backed property will
never reflect a value fetched by an *earlier* install if this is a
fresh reinstall (since reinstall wipes `UserDefaults` entirely). See
`LESSONS_LEARNED.md` #9 for the full detail and the fix (mirror the
value into the Keychain on every successful fetch, read that mirror
first for any synchronous pre-sync check). This specifically applies to
a free-trial-cooldown value if the app has one; budget for it for any
synchronous, launch-time gate that reads a Remote-Config-tunable value.

## Testing without touching the real Firebase project

`RemoteConfiguring` should be a real Domain protocol with a fake
implementation for unit tests (`MockRemoteConfiguring` or equivalent) —
see `ARCHITECTURE.md`'s Testing section. The concrete Firebase-backed
service itself is deliberately *not* unit-tested (same boundary
reasoning as any other third-party SDK wrapper); verify it manually by
actually changing a value in the Firebase console and confirming the
running app picks it up on next sync.
