# [FILL IN: App Name]

> This is a template for the **new app's own** `README.md` (setup/run/
> test instructions for a developer working in that repo) — not to be
> confused with `iOSAppStarterKit/README.md`, which documents this kit
> itself. Copy this file to the new repo's root as `README.md` and fill
> in every `[FILL IN]` marker.

## [FILL IN: one-paragraph description of what the app does]

## Device / platform support

[FILL IN: Universal (iPhone+iPad)? iPhone only? Other platforms? — see
`ARCHITECTURE.md`'s "Device / platform support" for the mechanism.]

## Testing subscriptions locally

The app gates [FILL IN: core feature] behind a free-usage allowance (see
`ARCHITECTURE.md`'s [FILL IN: usage-limit section name]), backed by an
auto-renewable subscription purchased through the App Store. The
`[FILL IN: App Name]` scheme is wired to a local StoreKit Testing
configuration (`[FILL IN: App Name]/Resources/Subscription.storekit`),
so purchase, renewal, and revocation/expiration can all be exercised —
in Simulator or on a real device — without an enrolled Apple Developer
Program account or a real App Store Connect product.

A paid Developer Program membership (and a matching subscription product
configured in App Store Connect, with the same product ID as
`[FILL IN: concrete SubscriptionManaging implementation]`) is only
required to actually ship the app — never for local development or
`xcodebuild test`.

**Practical notes, learned the hard way** (see
`iOSAppStarterKit/docs/subscription-paywall-pattern.md` for the full
detail):
- The local StoreKit config only attaches when Xcode itself launches the
  app for the `[FILL IN: App Name]` scheme (Run/Cmd+R). A plain `simctl
  launch` or an app reopened from the home screen after being
  backgrounded won't have it.
- Don't edit `Subscription.storekit`'s settings while the app is
  actively running — stop the app first, change the setting, then Run
  again.
- **Known limitation**: restoring a purchase after deleting and
  reinstalling the app does not work locally — validating that scenario
  needs a real (or Apple Sandbox) Apple ID against a real App Store
  Connect product.

## Testing the free-trial reinstall cooldown

[FILL IN, if this app has a no-payment-info free trial — see
`ARCHITECTURE.md`'s reinstall-abuse-mitigation section]:

1. **Blocked path** (the common case, needs no Remote Config change):
   install the app, delete it, reinstall within `free_trial_cooldown_days`
   (10 days by default) — the paywall should show immediately, with no
   free allowance granted.
2. **Elapsed-cooldown path**: set `free_trial_cooldown_days` to `0` in
   the Firebase console, launch the app once (any install) and leave it
   running a few seconds so the sync picks up and Keychain-caches the
   change, *then* delete and reinstall — the fresh install should be
   granted normally. Revert the Remote Config value afterward.
3. Check New Relic for the relevant analytics event with an outcome
   indicating a blocked trial replay, on the blocked path.

## Testing the forced-update kill switch

No Apple Developer Program enrollment needed — this doesn't touch
StoreKit at all, so it works at any signing tier, in Simulator or on a
real device.

| Parameter key | Type | Purpose |
|---|---|---|
| `minimum_app_version` | String | Lowest version still allowed to run |
| `app_store_url` | String | Where "Update Now" sends a blocked user |

To test: set `minimum_app_version` above the current build's version in
the Firebase console, then either relaunch or background/foreground the
app. Confirm the block screen appears with no way to dismiss it, and
that "Update Now" opens `app_store_url`. Set `minimum_app_version` back
down afterward and confirm normal use resumes.

## [FILL IN: delete this section if the app has no post-trial fair-use metering] Testing fair-use usage throttling

See `docs/subscription-paywall-pattern.md`'s "Fair-use throttling for
post-trial/subscribed usage" for the mechanism.

| Parameter key | Type | Purpose |
|---|---|---|
| `[FILL IN: soft-threshold parameter name]` | Int (seconds) | "Soft" threshold — matches whatever's disclosed in the Terms of Use |
| `[FILL IN: hard-threshold parameter name]` | Int (seconds) | "Hard" threshold, past which throttling increases further — never a block |
| `[FILL IN: soft-delay parameter name]` | Int (seconds) | Extra delay once over the soft threshold |
| `[FILL IN: hard-delay parameter name]` | Int (seconds) | Extra delay once over the hard threshold — replaces, not adds to, the soft delay |

To test: set the soft threshold low (e.g. `10`) in the Firebase console,
background/foreground the app to pick up the new value, then use the
metered feature past that total. Confirm a delay appears before the
result is delivered, the status text reflects waiting (not "in
progress"/"done"), and a new unit of work can't be started during the
delay. Lower the hard threshold too and confirm the delay increases
further — and that the feature is still never blocked, even well past
the hard threshold. Revert both values afterward. To confirm the
analytics side: background the app (the flush only happens on
backgrounding, not on a timer) and check for the usage-metering event
with the expected quantity/subscription-status attributes, and a
`userId` that stays stable across relaunches.

## Legal clickwrap + consent

See `ARCHITECTURE.md`'s "Legal clickwrap + consent" for the mechanism.

1. A fresh install should show the legal-acceptance screen before
   anything else (after the kill-switch check). The checkbox starts
   unchecked and the continue button stays disabled until checked.
2. [FILL IN, if this app has an additional one-time consent gate —
   e.g. recording/capture consent]: confirm it appears once, before the
   relevant feature is first used, and does not reappear on subsequent
   uses.
3. To test forced re-acceptance, bump a `LegalDocumentVersion` constant
   and relaunch — the relevant gate should reappear.
4. Check New Relic for the relevant analytics event on acceptance.

### Hosting the legal pages

The full Terms of Use, Privacy Policy, and Support text live in
`public/terms.html`, `public/privacy.html`, and `public/support.html`
at the repo root (not bundled in the app), deployed via Firebase
Hosting on this app's Firebase project:

```
firebase deploy --only hosting
```

Live at:
- `https://[FILL IN: firebase project ID].web.app/terms.html`
- `https://[FILL IN: firebase project ID].web.app/privacy.html`
- `https://[FILL IN: firebase project ID].web.app/support.html`

The Terms/Privacy text needs a final legal review before shipping — see
`iOSAppStarterKit/docs/legal-compliance-checklist.md`. Also currently
scoped to `[FILL IN: launch jurisdiction(s)]` only; revisit before
expanding elsewhere.
