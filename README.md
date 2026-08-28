# iOS App Starter Kit

A reusable blueprint for kicking off a new subscription-based iOS app —
SwiftUI, Firebase Remote Config, New Relic monitoring, StoreKit 2
subscriptions, App Store Connect CLI automation — without re-learning a
long list of real, hard-won gotchas from scratch.

This is not a code template you `git clone` and run — there's no
universal Xcode project here, because the actual product logic (Domain
models, Data services, Views) is inherently specific to whatever your
app actually does. What *is* reusable, and what this kit packages, is
everything **around** the product logic: the engineering standards, the
layered architecture shape, the third-party infrastructure setup steps,
and — most importantly — a concentrated list of specific, non-obvious
gotchas that cost real debugging time building a production app of this
shape, so you don't have to rediscover them independently.

It doesn't matter what your app's core feature actually is — voice
translation, photo-to-video generation, any other cloud-AI-backed
capability — the "baseline infrastructure" every subscription-gated,
Firebase/New-Relic-backed App Store app needs is the same, and that's
what this kit covers.

## What's in here

| File | Purpose |
|---|---|
| `CLAUDE.md.template` | Drop-in `CLAUDE.md` for the new repo — engineering standards, testing philosophy, git process, pre-merge checklist. Copy to the new repo root as `CLAUDE.md` and fill in the `[FILL IN]` markers. |
| `ARCHITECTURE.md.template` | Drop-in `ARCHITECTURE.md` — the Domain/Data/Presentation layering, composition root pattern, and a skeleton for each "baseline" feature (paywall, kill switch, legal clickwrap) with placeholders for your product's specifics. |
| `LESSONS_LEARNED.md` | **Read this first.** Every hard-won, non-obvious gotcha worth knowing before writing the equivalent code — XcodeGen quirks, App Store Connect API automation quirks, StoreKit Testing quirks, device-idiom bugs, disk-space traps, cost-modeling method. Every item here is a real bug/rejection/failure that actually happened building an app of this shape, not a hypothetical. |
| `project.yml.template` | XcodeGen project config, pre-loaded with the `info.properties` fixes for two real Info.plist bugs (a key silently reverting on every `xcodegen generate`, and a missing orientation declaration that gets an upload rejected). |
| `APP_README.template.md` | Drop-in `README.md` for the new repo itself — setup/testing instructions (StoreKit Testing quirks, kill-switch testing steps, legal-page hosting). |
| `docs/firebase-remote-config-setup.md` | Step-by-step Firebase project setup + the sync pattern (fetch on cold launch + foreground resume, fail open to last-known-good) + a concrete baseline parameter set with the business logic behind each one. |
| `docs/new-relic-analytics-setup.md` | Step-by-step New Relic setup + a specific, reasoned feature-flag configuration + a baseline custom-event taxonomy to build on, including the batch-on-backgrounding usage-metering pattern. |
| `docs/new-relic-cli-automation.md` | Querying New Relic (NRQL) and building dashboards from the command line via the NerdGraph API — credential storage, a reusable query script, and real GraphQL gotchas. |
| `docs/subscription-paywall-pattern.md` | The free-trial/paywall/StoreKit 2 design — usage metering, mid-session cutoff, reinstall-abuse mitigation, fair-use throttling for post-trial usage, and three StoreKit-API false starts distilled into "use this one, skip the other two." |
| `docs/app-store-connect-cli-automation.md` | How to fully automate archive → export → TestFlight upload → beta review submission from the command line, including every real gotcha (auth propagation delay, manual-signing export options, the two-step beta-review API call). |
| `docs/legal-compliance-checklist.md` | What a Terms/Privacy/consent flow needs to cover for an app like this, common gaps in a first AI-drafted legal pack, and what still needs a real lawyer regardless of how complete a template looks. |
| `legal/terms.html.template`, `privacy.html.template`, `support.html.template` | Hosted-page templates, genericized with `[FILL IN]` placeholders, structurally intact (numbered sections, fair-use clause, contact section). |
| `LICENSE` | MIT — copy, modify, and reuse any or all of this freely, including commercially. |

## How to use this for a new app

1. Read `LESSONS_LEARNED.md` end to end before writing any code — most items there are cheaper to build in from day one than to retrofit.
2. Copy `CLAUDE.md.template` → new repo's `CLAUDE.md`, `ARCHITECTURE.md.template` → new repo's `ARCHITECTURE.md`, `APP_README.template.md` → new repo's `README.md`; fill in every `[FILL IN: ...]` marker with your app's specifics.
3. Copy `project.yml.template` → new repo's `project.yml`, adjust the app name/bundle ID/dependencies, keep the `info.properties` block's structure (that's the part that isn't obvious to reinvent).
4. Follow `docs/firebase-remote-config-setup.md` and `docs/new-relic-analytics-setup.md` to stand up both third-party services before writing your `SettingsStore`/`AppDelegate` equivalents.
5. Adapt `docs/subscription-paywall-pattern.md` and the `legal/*.template` files once your app's actual monetization/legal model is decided — the specific decisions documented (e.g. a true no-payment-info trial) are examples, not universal defaults; re-confirm each one explicitly for your product rather than copying it blind. Run the result past `docs/legal-compliance-checklist.md` before publishing, and get a real lawyer to review before shipping.
6. When you're ready to ship a TestFlight build, `docs/app-store-connect-cli-automation.md` has the exact working command sequence.

## Using this with Claude Code on a new project

If you're starting a new app's repo with Claude Code: point it at this
kit's `LESSONS_LEARNED.md` and `ARCHITECTURE.md.template` early in the
project, before much architecture gets decided — a fresh Claude Code
session in a new repo has no memory of anything in this kit unless you
show it. Copying the template files into the new repo (as step 2 above
describes) is what makes the patterns available to both you and any
future Claude Code session working in that repo.
