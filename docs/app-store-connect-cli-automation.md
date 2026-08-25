# App Store Connect CLI automation

The complete, tested-for-real working sequence to archive, export,
upload to TestFlight, and submit for external beta review — entirely
from the command line, no Xcode Organizer UI needed. Every step below
was actually run successfully end-to-end on a real production app;
every gotcha noted was hit for real, not anticipated in the abstract.

## Prerequisites

1. **An App Store Connect API key**, not an Apple ID/password (Apple ID
   auth via `altool` is deprecated and needs interactive password entry
   anyway, which doesn't work in a non-interactive automation context).
   Generate one at App Store Connect → **Users and Access** →
   **Integrations** → **App Store Connect API** (needs Admin access to
   the team):
   - Role: **App Manager** is sufficient for uploading builds — no need
     for Admin.
   - Download the `.p8` private key file **immediately** — App Store
     Connect only allows the download once, ever.
   - Note the **Key ID** (also embedded in the downloaded filename,
     `AuthKey_<KEY_ID>.p8`) and the **Issuer ID** (a UUID shown on the
     same page, above the keys table).
2. **A Release-signing setup that already works**, i.e. `xcodebuild
   archive` with `-configuration Release` succeeds and produces a
   correctly-signed `.app` — this doc assumes signing already works
   (see `project.yml.template`'s manual-signing fallback if Automatic
   signing fails to create/match a distribution profile) and focuses on
   the archive → upload → TestFlight steps after that.

## Step 1 — place the key where `xcodebuild` reliably finds it

```bash
mkdir -p ~/.appstoreconnect/private_keys
cp /path/to/AuthKey_<KEY_ID>.p8 ~/.appstoreconnect/private_keys/
```

**Gotcha hit for real**: passing `-authenticationKeyPath
/arbitrary/path/AuthKey_X.p8` directly to `xcodebuild -exportArchive`
produced a `401 NOT_AUTHORIZED` even with correct credentials. Placing
the key in the standard `~/.appstoreconnect/private_keys/` directory and
omitting `-authenticationKeyPath` (letting `xcodebuild` discover it by
Key ID alone) is what actually worked.

**If you get a 401 with a freshly-created key**: this is very likely
just propagation delay, not a real credential problem. A freshly
generated App Store Connect API key can take a few minutes to become
valid across Apple's auth servers. To isolate "is this a credential
problem or a propagation-delay problem" before assuming the key/IDs are
wrong, test the key directly against the API with a hand-built JWT
(ES256, no external dependencies needed beyond `openssl` + Python's
standard library):

```python
import base64, json, subprocess, sys, time

KEY_PATH = "/path/to/AuthKey_XXXXXXXXXX.p8"
KEY_ID = "XXXXXXXXXX"
ISSUER_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()

def der_to_raw_sig(der: bytes, order_len: int = 32) -> bytes:
    assert der[0] == 0x30
    idx = 2 if der[1] < 0x80 else 2 + (der[1] & 0x7F)
    def read_int(i):
        assert der[i] == 0x02
        length = der[i + 1]
        start = i + 2
        val = der[start:start + length].lstrip(b"\x00").rjust(order_len, b"\x00")
        return val, start + length
    r, idx = read_int(idx)
    s, idx = read_int(idx)
    return r + s

header = {"alg": "ES256", "kid": KEY_ID, "typ": "JWT"}
now = int(time.time())
payload = {"iss": ISSUER_ID, "iat": now, "exp": now + 600, "aud": "appstoreconnect-v1"}
signing_input = b64url(json.dumps(header, separators=(",", ":")).encode()) + "." + \
                b64url(json.dumps(payload, separators=(",", ":")).encode())
proc = subprocess.run(["openssl", "dgst", "-sha256", "-sign", KEY_PATH],
                       input=signing_input.encode(), capture_output=True)
jwt = signing_input + "." + b64url(der_to_raw_sig(proc.stdout))
print(jwt)
```

```bash
JWT=$(python3 make_jwt.py)
curl -s -H "Authorization: Bearer $JWT" "https://api.appstoreconnect.apple.com/v1/apps"
```

If this returns your app data, the key/IDs are correct and any
`xcodebuild` failure is tooling-specific (see the key-placement gotcha
above), not a credential problem — don't waste time re-checking the Key
ID/Issuer ID once this succeeds.

## Step 2 — archive

```bash
xcodegen generate   # if using XcodeGen — regenerate first, always
rm -rf build
xcodebuild -project YourApp.xcodeproj -scheme YourApp \
  -configuration Release -archivePath build/YourApp.xcarchive archive
```

## Step 3 — export options

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store-connect</string>
    <key>destination</key>
    <string>upload</string>
    <key>teamID</key>
    <string>[FILL IN: Team ID]</string>
    <key>signingStyle</key>
    <string>manual</string>
    <key>provisioningProfiles</key>
    <dict>
        <key>[FILL IN: bundle ID]</key>
        <string>[FILL IN: exact provisioning profile name]</string>
    </dict>
</dict>
</plist>
```

**Gotcha hit for real**: with `signingStyle: manual` and no
`provisioningProfiles` map, export failed with `"YourApp.app" requires a
provisioning profile` — even though the archive itself was correctly
signed. Manual signing style requires this explicit bundle-ID → profile-
name mapping in the export options; it will not infer it from the
archive.

## Step 4 — export + upload (one command does both, given `destination: upload`)

```bash
xcodebuild -exportArchive \
  -archivePath build/YourApp.xcarchive \
  -exportPath build/export \
  -exportOptionsPlist build/ExportOptions.plist \
  -authenticationKeyID XXXXXXXXXX \
  -authenticationKeyIssuerID xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

A successful run prints a progress log ending in `Upload succeeded.` /
`** EXPORT SUCCEEDED **`.

## Step 5 — bump the build number for every new upload, remember to redeploy the fix from `LESSONS_LEARNED.md` #1 if it reverts

`CFBundleVersion` must be strictly incremented for every new App Store
Connect upload under the same `CFBundleShortVersionString` — a re-upload
of a build number Apple has already seen is rejected outright. Bumping
the marketing version (`CFBundleShortVersionString`) is what triggers a
**new full App Review**; bumping only the build number does not (it's
what lets you push a TestFlight-only fix without waiting through a full
review again).

## Step 6 — confirm the build actually landed

```bash
curl -s -H "Authorization: Bearer $JWT" \
  "https://api.appstoreconnect.apple.com/v1/apps/<APP_ID>/builds?limit=5" \
  | python3 -c "import json,sys
for b in json.load(sys.stdin)['data']:
    print(b['id'], b['attributes']['version'], b['attributes']['processingState'])"
```

Processing can take anywhere from a couple of minutes to longer; poll
rather than assuming it's instant. Note: `curl` needs `-g` (disable URL
glob expansion) if you ever add a bracketed query filter like
`?filter[build]=...` — otherwise it fails with an opaque exit code.

## Step 7 — internal testers get it automatically; external testers need two explicit API calls

**Internal Testers groups get every new build automatically** — no
review, nothing further to do.

**External Testers groups do not** — adding a build to an external
group does **not** by itself trigger beta review; you need an explicit
second call:

```bash
# 1. Add the build to the external group
curl -s -X POST \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"data":[{"type":"betaGroups","id":"<EXTERNAL_GROUP_ID>"}]}' \
  "https://api.appstoreconnect.apple.com/v1/builds/<BUILD_ID>/relationships/betaGroups"

# 2. Explicitly submit for beta review (this is the step that's easy to
#    miss — adding to the group alone leaves it silently un-submitted)
curl -s -X POST \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"data":{"type":"betaAppReviewSubmissions","relationships":{"build":{"data":{"type":"builds","id":"<BUILD_ID>"}}}}}' \
  "https://api.appstoreconnect.apple.com/v1/betaAppReviewSubmissions"
```

Poll `betaReviewState` on the created submission (`WAITING_FOR_REVIEW` →
`APPROVED`/`REJECTED`). In practice, a build update under an
**already-approved marketing version** tends to clear much faster than a
first-ever submission — likely lighter-weight/automated review for that
specific case — but this isn't a documented guarantee from Apple, so
don't assume a specific turnaround time.

## Step 8 — set "What to Test" notes

```bash
# Find the (often auto-created, empty) localization first:
curl -s -H "Authorization: Bearer $JWT" \
  "https://api.appstoreconnect.apple.com/v1/builds/<BUILD_ID>/betaBuildLocalizations"

# Then PATCH it:
curl -s -X PATCH \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"data":{"type":"betaBuildLocalizations","id":"<LOCALIZATION_ID>","attributes":{"whatsNew":"..."}}}' \
  "https://api.appstoreconnect.apple.com/v1/betaBuildLocalizations/<LOCALIZATION_ID>"
```

## Resubmitting a *rejected* app version for full App Store review

Different from the TestFlight beta-review flow in Steps 7-8 above — for
App Store review, a rejection leaves the app version, the subscription,
and the subscription group all as items inside the *same*
`reviewSubmission` (state `UNRESOLVED_ISSUES`), each individually
either `READY_FOR_REVIEW` or `REJECTED`. Once you've fixed the app and
uploaded a new build:

```bash
# 1. Attach the new build to the app version:
curl -s -X PATCH \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"data":{"type":"builds","id":"<NEW_BUILD_ID>"}}' \
  "https://api.appstoreconnect.apple.com/v1/appStoreVersions/<VERSION_ID>/relationships/build"

# 2. Try resubmitting the *original* reviewSubmission (not a new one):
curl -s -X PATCH \
  -H "Authorization: Bearer $JWT" -H "Content-Type: application/json" \
  -d '{"data":{"type":"reviewSubmissions","id":"<ORIGINAL_SUBMISSION_ID>","attributes":{"submitted":true}}}' \
  "https://api.appstoreconnect.apple.com/v1/reviewSubmissions/<ORIGINAL_SUBMISSION_ID>"
```

**Gotcha hit for real, unresolved on the API side**: creating a *new*
`reviewSubmission` to resubmit fails outright — the app version stays
claimed by the original submission (`STATE_ERROR.ITEM_PART_OF_ANOTHER_SUBMISSION`),
and `reviewSubmissions` doesn't support `DELETE` via the API at all
(only CREATE/GET/UPDATE) — an accidentally-created empty submission can
only be discarded from the **web UI**, not undone via the API. Resubmit
the *original* submission instead (step 2 above).

That PATCH itself then kept failing for 20+ minutes with `409
STATE_ERROR: "Version is not ready to be submitted yet, please try
again later"`, even with the new build correctly attached and `VALID`.
**What actually worked**: not waiting longer, and not a Resolution
Center reply — the App Store Connect **web UI** has an **"Edit" button
directly on the specific rejected item** (in the submission's "Items
Submitted" list). Opening it, confirming the new build is attached
inside that view, and clicking the page's own "Update Review" button
resolved it immediately. The likely explanation: the top-level PATCH
above only ever touches the `reviewSubmission`'s own `submitted` flag —
it never transitions the individual `REJECTED` `reviewSubmissionItem`
for the app version. The UI's per-item edit flow evidently does that
transition as part of confirming the new build; **the public API does
not appear to expose an equivalent per-item action**. If the
resubmission PATCH keeps 409ing after a build swap, stop polling it and
resubmit through the web UI's per-item "Edit" affordance instead.

## Environment gotchas encountered doing all of the above

See `LESSONS_LEARNED.md` #15 for the full list (disk space exhaustion
from `DerivedData`/Simulator caches, a stale `NODE_OPTIONS` breaking the
`firebase` CLI, curl's bracket-globbing issue) — all of these can
surface *during* this exact workflow and look like unrelated failures
if you don't already know to check them.
