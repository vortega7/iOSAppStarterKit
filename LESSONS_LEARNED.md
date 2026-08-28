# Lessons Learned

Non-obvious gotchas worth knowing before writing the equivalent code —
SwiftUI, XcodeGen, Firebase Remote Config, New Relic, StoreKit 2
subscriptions, no backend. Every item here reflects a real failure mode
(a build rejection, a production bug, a broken test suite, a stuck
tool), not a hypothetical — read this before implementing the
equivalent feature; it's cheaper to build these in from day one than to
retrofit them after a real user or a real App Store Connect upload
finds the gap.

---

## 1. XcodeGen + Info.plist: declare every key that matters in `info.properties`, never rely on the raw file alone

**Rule**: with `GENERATE_INFOPLIST_FILE: NO` and `info.path` pointing at
a checked-in `Info.plist`, do not treat a direct edit to that raw file
as durable for any key that actually matters. `xcodegen generate` does
not reliably preserve arbitrary edits made only to the base file —
declare the key inside `project.yml`'s `info.properties` block instead,
which `xcodegen generate` re-applies on every run.

**Verify**: run `xcodegen generate` **twice in a row** and diff
`Info.plist`. If the second run changes anything you just set outside
`info.properties`, it isn't actually sourced from there.

**Keys known to need this treatment**: `ITSAppUsesNonExemptEncryption`,
`CFBundleVersion`, `CFBundleShortVersionString`. Treat any future
build-relevant Info.plist key the same way by default.

---

## 2. Declare `UISupportedInterfaceOrientations` the moment the app supports more than one fixed orientation

**Rule**: the moment an app's orientation-lock delegate method can
return anything other than a single fixed orientation mask, declare
`UISupportedInterfaceOrientations` (and `UISupportedInterfaceOrientations~ipad`
if Universal) in `info.properties`, listing every orientation the app
might *ever* return. A portrait-only app doesn't need this declared at
all; an app that supports even one runtime-togglable orientation does —
App Store Connect rejects the upload outright otherwise ("No
orientations were specified... To support iPad multitasking,
specify..."), even if the runtime orientation-lock logic itself is
correct. The static declaration and the runtime logic are two separate
mechanisms — having one correct does not satisfy the other.

---

## 3. Device-specific behavior must check the device idiom directly, never infer it from measured geometry

**Rule**: any change meant to be device-specific (e.g. "iPad only," not
"landscape only") needs an explicit device-idiom check
(`UIDevice.current.userInterfaceIdiom == .pad` on iOS/iPadOS), separate
from and alongside any orientation/size-class check. A compact/regular
size-class or `GeometryReader`-derived "is this a small width" check
answers "how much room do I have right now"; a device-idiom check
answers "which hardware is this" — they are different questions, and a
request scoped to one device answers the second, never the first. Using
a geometry signal as a stand-in for device identity silently applies a
device-scoped change to every device that happens to share that
geometry (e.g. one phone's landscape width matching a tablet's portrait
width).

**Verify**: grep every touched layout constant for a bare
`isCompact ? X : Y`-style ternary that changed value in a device-scoped
change, and confirm each one is *also* gated on the device-idiom check.
Add a regression test asserting the **unaffected** device/orientation's
value is unchanged — not just that the target device's new value is
correct; the point is proving the unaffected case is provably
unaffected. Verify visually on the unaffected device/orientation too,
not just the target one — overflow/clipping regressions are easy to
miss in a screenshot review that only looks at the changed target.

---

## 4. Support every targeted platform from day one, not as a later retrofit

**Rule**: design and verify every target platform (iPhone, iPad, other
Apple platforms) up front for every feature, rather than shipping one
platform first and adding another later. A multi-round platform retrofit
onto an already-shipped, single-platform app is exactly the situation
that produces the class of regression in #3 — the second platform's
"just bump this value" changes have no natural way to stay scoped to
the second platform unless the idiom-check discipline above is applied
from the start.

**Portability note**: a 100%-SwiftUI-views codebase (no
`UIViewRepresentable`, no `UIView`/`UIViewController` subclasses) is
necessary but not sufficient for genuinely easy multi-platform support.
Two real constraints remain even with a pure-SwiftUI view layer: (a)
whether your third-party SDKs actually support the target platform —
check their current platform support matrix, don't assume; and (b) core
interaction patterns (a press-and-hold gesture, a split-screen layout)
usually need real per-platform UX redesign once you move beyond
iPhone/iPad into something like watchOS or visionOS, not just
recompilation.

---

## 5. StoreKit 2 subscription status: read `Transaction.all` directly, don't trust the higher-level "current entitlement" APIs at face value

**Rule**: for computing "is this subscription currently active," prefer
iterating `Transaction.all` and computing entitlement from a matching
transaction's own `revocationDate`/`expirationDate` directly, over
`Transaction.currentEntitlements` or `Product.SubscriptionInfo.status`.
Both higher-level APIs can, in real on-device StoreKit Testing, report
no entitlement for a transaction that Xcode's own Transaction Manager
confirms is active and unrevoked. Budget real on-device purchase/
refund/relaunch testing for any subscription-status code — StoreKit 2's
documented API shape does not reliably match on-device behavior in
every case.

**Known StoreKit Testing limitation** (not a bug in your code): after
deleting and reinstalling an app locally, "Restore Purchases" against a
**local StoreKit Testing** configuration will never recover the prior
transaction, regardless of which status-read API is used —
`AppStore.sync()` completes without error, but the entitlement is never
found. A local test transaction appears to be tied to the app's
installed container, not to a simulated account the way a real purchase
is tied to a real Apple ID on Apple's servers. Don't spend further time
chasing this specific scenario locally; validating real restore
requires a real (or Apple Sandbox) Apple ID against a real App Store
Connect product.

---

## 6. De-duplicate `Transaction.updates` by transaction ID, persisted across launches

**Rule**: subscribing to StoreKit's `Transaction.updates` stream
replays the current/recent transactions on every fresh subscription to
it — in practice, on every app launch — not just genuinely new events.
Any code consuming this stream for analytics/lifecycle logging needs a
persisted (survives relaunch) set of already-processed transaction IDs,
checked before logging anything from the stream, or a single real event
(e.g. one revocation) gets re-logged on every subsequent launch.

---

## 7. A continuously-foregrounded session needs its own subscription-status re-check timer

**Rule**: neither a background/foreground transition nor
`Transaction.updates` is guaranteed to promptly surface a subscription
revocation for a user who keeps the app open and foregrounded
continuously. Add a third, independent trigger — a periodic re-check
timer (every few minutes) — alongside foreground-resume and the
transaction stream; each of the three catches a case the other two
miss.

---

## 8. Any `try?`/silent-failure around a StoreKit or paywall call needs an explicit failure UI state

**Rule**: don't let a product-load or purchase-flow failure resolve to
an indefinite loading spinner with no way forward. Treat every
`try?`-style silent failure around a subscription-related call as a
place to add an explicit failed state with a retry affordance — a
silently swallowed error there is a common way to strand a user with no
Subscribe button and no way to recover without force-quitting.

---

## 9. A synchronous, pre-config-sync check needs its value mirrored into the Keychain, not just UserDefaults

**Rule**: any value that (a) needs to be read **synchronously**, very
early in launch — before that launch's own async remote-config fetch
can possibly complete — **and** (b) needs to **survive a reinstall**,
cannot be satisfied by `UserDefaults` alone. A reinstall wipes
`UserDefaults` entirely, so the value is always back at its hardcoded
default immediately after reinstall, regardless of what a *previous*
install had successfully synced from remote config. Fix: mirror the
fetched value into the Keychain (which does survive app deletion — only
a full device erase clears it) on every successful fetch, and have the
synchronous, early check read the Keychain-cached mirror first, falling
back to `UserDefaults`/the hardcoded default only if no install on this
device has ever synced a value before. This is a general pattern for
any future remote-config-tunable value with both properties above, not
a one-off special case.

---

## 10. A "fresh install" signal needs its own dedicated flag, not an inferred one

**Rule**: don't infer "this is a fresh install" from the absence of
some other key that's only written once a user takes a specific action.
A user who installs, does nothing, and simply reopens the app later
(no reinstall at all) will incorrectly look "fresh" again on that
second launch if freshness is inferred that way. Use a dedicated flag,
set `true` immediately once a fresh-install check resolves (regardless
of outcome), and key all future "is this fresh" logic off that flag
specifically.

---

## 11. A test helper's injectable-dependency default must fall back to a fake, never the real implementation

**Rule**: any test helper constructing a type with an injectable
dependency (credential store, analytics, remote config, network client,
etc.) should either have no default value at all, or default to a
**fake** — never to the real, side-effecting implementation. A default
that silently falls back to the real Keychain/network/filesystem will
eventually cause one test's state to leak into and break unrelated
later tests, in a way that's expensive to trace back to the missing
explicit argument.

---

## 12. Client-side entitlement gating is a UX layer, not real enforcement — treat backend enforcement as a separate, deliberate decision

**Rule**: client-side checks (a usage counter, a cached subscription
status) can be bypassed by a technically capable user, especially in an
app with no backend where the client calls a paid third-party API
directly with an embedded key. Recognize this as a known, accepted
limitation rather than something client-side mechanisms are meant to
fully solve. Real enforcement needs a backend that proxies the actual
paid API calls and validates entitlement server-side, plus a webhook
subscription to the platform's server-to-server purchase notifications
(e.g. Apple's App Store Server Notifications V2) for revocation events
that don't depend on the client app ever running again. Scope this as
its own architecture decision — a new backend service, an auth/user-
identity model, a hosting decision — not something to bolt onto a
client-side MVP incrementally.

---

## 13. Verify current cloud-API list pricing directly; don't estimate from memory when sizing a usage cap

**Rule**: when sizing a fair-use/usage-limit threshold against
subscription pricing, pull the actual current list prices for the
*exact* API tier the app's code calls (verify from the actual
request-building code — confirm whether it's Standard vs. Enhanced/
Neural/HD, Basic vs. Advanced API version, whether a quality/model flag
is explicitly set) rather than estimating from training-data memory,
since prices and tier structures change over time. Compute a
**marginal cost per unit of usage** (ignore pooled free tiers once the
app has any meaningful number of users sharing a billing account/
project — those get exhausted quickly at scale), give a **sensitivity
range** rather than one falsely-precise number (the biggest uncertainty
is usually a real-world usage-density assumption), and use that range
to sanity-check any usage cap before publishing it in a Terms of
Service or enforcing it in code.

---

## 14. See `docs/app-store-connect-cli-automation.md` for the full CLI-automation gotcha list

Auth propagation delay on a freshly created API key, the standard
`~/.appstoreconnect/private_keys/` directory requirement, the
`provisioningProfiles` map required for manual signing style in
`ExportOptions.plist`, the non-obvious fact that adding a build to an
External Testers group does **not** itself trigger beta review (a
second, explicit API call is required), and — resubmitting a
*rejected* App Store review — creating a new `reviewSubmission` fails
outright (the app version stays claimed by the original one, which
can't be deleted via the API), and even resubmitting the *original*
submission via `PATCH .../reviewSubmissions/{id}` can 409 indefinitely
after a build swap until you use the App Store Connect **web UI**'s
per-item "Edit" affordance on the specific rejected item — the public
API doesn't appear to expose an equivalent per-item action.

---

## 16. Auto-renewable subscriptions: two App Store Review requirements to build in from day one, not retrofit after a rejection

**Rule**: the subscription's title, length, price, and functional links
to the Privacy Policy *and* Terms of Use must be visible **directly in
the purchase flow screen itself** — a Settings "Legal" section with the
same links elsewhere in the app does not satisfy this (Guideline
3.1.2(c)). If the app uses Apple's Standard EULA (no Custom EULA
configured), the App Store Description also needs an actual clickable
Terms of Use URL — prose like "see our Terms" with no URL does not
count as a functional link.

**Rule**: the purchase flow must be reachable **without needing to
exhaust any usage limit first** (Guideline 2.1(b)) — App Review needs
to locate and test the in-app purchase in a reasonable session. A
paywall that only appears reactively, once a free allowance runs out,
is not practically testable. Add a direct entry point (e.g. a
"Subscribe" row in Settings, shown whenever the user isn't already
subscribed) from the first version that ships a paywall.

Both are pure UI additions, not changes to entitlement/limit-checking
logic — cheap to build in from the start, and a full review-cycle cost
to retrofit after a rejection (confirmed the hard way: a real submission
was rejected for both of these at once). See
`docs/subscription-paywall-pattern.md`'s "App Store Review requirements
for the purchase flow" for the full detail.

---

## 15. Environment/tooling gotchas worth knowing ahead of time

- **Disk space**: `DerivedData` and its SPM package-checkout cache can
  silently grow large enough to exhaust available disk space, at which
  point `git`, `xcodebuild`, and package resolution all start failing
  with unrelated-looking errors (`unable to write new index file`,
  `NSPOSIXErrorDomain`). Clearing `DerivedData` is always safe (pure
  build cache, regenerated automatically). Old/unused Simulator device
  data is also safe to prune and can be tens of GB — check disk space
  before assuming a build failure is a real code problem.
- **A Node-based CLI failing with `MODULE_NOT_FOUND` on some unrelated
  preload script**: check for a stale `NODE_OPTIONS` environment
  variable pointing at a temp file that no longer exists. Work around
  per-command by unsetting it for that invocation.
- **`curl` with a literal `[`/`]` in a query string** (e.g.
  `?filter[build]=...`) needs glob expansion disabled, or it fails with
  an opaque exit code.

---

## 17. Every test file that constructs a store must reset *all* of its `UserDefaults`-backed keys, not just the ones the current test touches

Hit three separate times building one feature (a monthly usage counter +
two new tunable settings on an existing store): adding a new
`UserDefaults`-backed property to a store, then writing a test that sets
it to a test value, silently broke *other, unrelated* tests in *other*
test files — because `UserDefaults.standard` is real and shared across
the whole test process, not per-test-isolated, and those other files'
"fresh install" helper only reset the keys that existed *when that
helper was written*.

**Symptom**: a test that passed in isolation fails only when the full
suite runs, with a value that looks like it leaked from nowhere (e.g. a
`pipelineState` stuck in a throttled/waiting state in a test that never
touches throttling at all).

**Rule**: every test-file helper that constructs a store backed by
`UserDefaults` must `removeObject(forKey:)` for **every** key that
store persists, not only the keys that file's own tests set — even keys
introduced by a completely different feature/PR. When you add a new
`UserDefaults`-backed property anywhere, grep for every other test
file's store-construction helper and add the same reset there, before
assuming "I reset it in my own test file" is sufficient. This is easy
to miss precisely because the helper you need to update usually looks
unrelated to the feature you're adding.

---

## 18. A `.sheet { }` closure's content needs its `@Observable` stores re-injected explicitly — non-optional `@Environment` reads that work fine on iPhone/iPad can crash on "app runs on Mac" (Catalyst/"Designed for iPad")

**Symptom hit for real, post-launch**: a build that worked correctly on
iPhone and iPad crashed on Apple Silicon Mac (`EXC_BREAKPOINT`, a Swift
`assertionFailure` inside `EnvironmentValues.subscript.getter`) the
moment any of the app's `.sheet { }`-presented views tried to read a
store via `@Environment(SomeStore.self)` (the non-optional form, which
traps hard if the value isn't present).

**Root cause**: those sheet-presented views relied on *inheriting* the
`.environment(...)` values injected further up the view tree (typically
at the composition root) rather than having them re-injected on the
sheet's own content. That inheritance holds across a normal SwiftUI
`.sheet` presentation on iPhone/iPad, but **not** across the
macOS/Catalyst sheet-presentation bridge (`SheetBridge`/
`PresentationHostingController`) — the sheet's content view hierarchy
there doesn't automatically inherit environment values the same way.

**Why this passes App Review anyway**: reviewers test on iPhone/iPad,
not Mac, so this class of bug is invisible to standard review even when
"supports Mac" is enabled in App Store Connect.

**Fix**: explicitly re-inject every store a sheet's content needs,
directly in the `.sheet { }` closure —
`SomeSheetView().environment(storeA).environment(storeB)` — rather than
relying on inheritance. Cheap and mechanical once identified; the cost
is entirely in noticing it, since the failure mode only reproduces on
an actual Mac (a real device or the "My Mac (Designed for iPad)" run
destination in Xcode — the plain iOS Simulator doesn't catch it).
**If "supports Mac" is enabled for a Universal/iPad-idiom app, verify
every sheet presentation on an actual Mac destination before shipping**,
not just iPhone/iPad — or disable Mac availability in App Store Connect
until that verification happens.
