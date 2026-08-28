# Legal compliance checklist

What a real legal review of an AI-drafted legal pack actually finds and
decides, kept as a checklist so the same review doesn't need to happen
from scratch on a new app — plus a hard reminder that **none of this is
a substitute for real counsel**.

## The clickwrap mechanism (architecture — see `ARCHITECTURE.md`)

A versioned acceptance gate, root-level (kill switch → legal acceptance
→ main content), unchecked-checkbox-required (not passive browsewrap).
Legal text hosted externally (Firebase Hosting or equivalent), not
bundled in the app binary, so it can be updated without an app release.
Bumping a version constant is what forces re-acceptance for existing
users — decide explicitly, per change, whether a given legal-text update
is material enough to warrant that. (Example of the tradeoff: choosing
*not* to force re-acceptance for a usage-limit clause addition because
the app hadn't launched publicly yet and there was no real user base to
disrupt — that reasoning stops holding the moment there are real
users.)

Any additional one-time consent specific to the app's core feature (see
`legal/terms.html.template` section 6's placeholder) should be its own
gate, shown once at first use of that feature, not re-shown per use
unless there's a specific legal reason to.

## Real gaps found in a first AI-drafted legal pack — check for these again

A first legal draft, whether from an AI tool or a generic template, can
have a solid skeleton but real, substantive gaps — check for these
specifically on review:

- **No named subprocessors** — a generic draft often doesn't name the
  actual third-party services the app uses. Decide explicitly whether
  the Privacy Policy names each one specifically (cloud AI provider,
  remote-config provider, analytics provider, etc.) or describes them
  generically ("a cloud translation-processing provider") — this is a
  GDPR/CCPA-style "categories of recipients" transparency judgment call,
  **not** something Apple's App Privacy questionnaire in App Store
  Connect requires either way (a claim worth correcting if you see it
  asserted elsewhere: that questionnaire — the "nutrition label" — asks
  about data *types and purposes* you collect, not vendor names, and is
  a separate mechanism from the Privacy Policy text entirely). Named
  vendors are common in voice/transcription apps specifically, since
  privacy-conscious users in that category tend to want to know exactly
  which infrastructure touches their voice recordings — but weigh that
  against any competitive-concealment concern for your product, and
  revisit the decision once the app has real, meaningful traffic.
- **Biometric/voiceprint law**, if the app processes voice or face data
  at all — e.g. Illinois' BIPA (private right of action, statutory
  damages per violation) wasn't addressed with any specificity in the
  first draft. Flag this explicitly for counsel review if the app
  captures/processes voice, face, or any other biometric identifier —
  don't assume a generic "we process audio" clause covers it.
- **Apple's Custom EULA mechanism** (a separate App Store Connect field,
  distinct from the in-app clickwrap) often isn't mentioned in a generic
  draft at all — it's a manual App Store Connect configuration step,
  separate from anything in the codebase, and easy to forget.
- **The App Store Description itself must not misrepresent data
  handling** — checked separately from the Privacy Policy/legal text,
  since App Review reads the public Description too. Hit for real: a
  Description claimed "your audio never leaves your phone" while the
  app actually sent it to a third-party cloud AI provider — an
  unrelated but real misrepresentation caught while fixing a different
  rejection. If App Review's questionnaire asks whether any data is
  shared with a third-party AI and the honest answer is yes, say so
  factually in the Notes/Resolution Center reply, and grep your own
  Description/marketing copy for any absolute claim ("never leaves your
  device," "100% private," "no data collected") before submitting —
  those are exactly the claims most likely to silently drift out of
  sync with what the app actually does as it evolves.

## Clauses worth pushing back on in a generic/aggressive draft

- **Consumer indemnification clauses** ("you agree to indemnify us for
  any claim arising from your use") are presumptively unfair under EU
  consumer law (even if the launch isn't EU-scoped yet) and are out of
  step with how major consumer apps in the same space write their
  terms — and are practically worthless against an individual consumer
  anyway. Consider cutting entirely rather than including reflexively.
- **Per-use/per-session re-consent** for an app whose core interaction
  is rapid and repeated (e.g. push-to-talk) breaks the product for
  little real legal benefit over a one-time, permanently-accessible
  notice. Weigh this specifically against the app's actual interaction
  model rather than defaulting to "more consent gates is always safer."
- **"Durable backend" acceptance logging** — a generic draft may assume
  backend infrastructure the app doesn't have. If there's no backend,
  an analytics custom event (see `docs/new-relic-analytics-setup.md`)
  is a workable v1 substitute for an acceptance audit trail — but it's
  explicitly weaker than a real backend log (bounded retention window)
  and should be flagged as such if a dispute-proof record is ever
  actually needed.

## Identity/contact decisions (a product decision, not a legal default)

- Whether the Terms name the developer directly or reference "the
  developer identified as the 'Seller' in this app's App Store listing"
  — note that Apple already mandates individual developer accounts
  display a real legal name as Seller, so the latter doesn't actually
  reduce disclosure, just changes where it's stated.
- Whether Contact sections point to a direct support email or the App
  Store support listing.
- **Governing law** — this needs real jurisdiction-specific counsel
  confirmation, not a default. An AI review can flag an open question
  (e.g. whether a specific province/state's "Internet agreement"
  supplier-disclosure requirements are satisfied by pointing at the App
  Store listing rather than stating supplier info directly in-document)
  but cannot resolve it — treat any such flag as a required follow-up
  with real counsel, not a settled answer.

## Scope discipline

Explicitly scope legal text to a specific launch jurisdiction (or set of
jurisdictions) rather than trying to write something globally correct
from the start — e.g. "this covers US/Canada only" stated plainly in the
document, with a note to revisit before expanding to the EU (GDPR
Article 27 representative requirement), Japan, or any other market with
materially different consumer/privacy law. A correct-for-two-countries
document beats an incorrect-for-everywhere one.

## Before actually publishing, every time

1. **Get the drafted text reviewed by real counsel** — an AI legal
   review (this one included) catches structural gaps and asks the
   right questions, but is not a substitute for a licensed opinion,
   especially on the governing-law and biometric-data points above.
2. **Configure Apple's Custom EULA field and the App Privacy
   questionnaire** in App Store Connect — neither is code, both are
   still required manual steps, easy to forget since nothing in the
   codebase reminds you.
3. **Cross-check the Privacy Policy's subprocessor list against the App
   Privacy questionnaire's actual answers** — these drift independently
   and Apple does not check that they match for you.
