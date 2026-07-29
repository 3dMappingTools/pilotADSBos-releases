# `latest.json` — the update manifest

**Fetch URL (stable, never changes):**

```
https://raw.githubusercontent.com/3dMappingTools/pilotADSBos-releases/main/latest.json
```

Use the `raw.githubusercontent.com` URL, **not** a release-asset URL. Asset URLs
contain the tag, so they change with every release — an app pinned to one would
stop seeing updates the moment we shipped the next version. This path is
permanent; only the file's contents change.

Expect up to ~5 minutes of CDN caching after a push. That is fine for a check
that runs at most once a day.

---

## What is in it today, and what is deliberately missing

`latest.json` currently carries **tier 2 only** — "a newer full image exists,
which needs reflashing a card."

**There is no `tier1` key, and that is intentional.** Tier-1 (over-the-air
replacement of the `dpcore` binary) is built and unit-tested on the OS side, but
it **ships dark**: every unit in the field has an all-zero placeholder signing
key, so `/update/status` reports `armed: false` and `/update/dpcore` returns 503.
The endpoint refuses *everything*, including a genuine update from us. That is
the intended safety posture — no code can be pushed to any unit until a real
production key exists.

Publishing a `tier1` block now would advertise an update the receiver is
guaranteed to reject, so it stays out until the key is baked and a signed
release exists.

**Apps must treat a missing `tier1` key as "no tier-1 update available" — never
as an error, and never as a reason to fall back to tier 2.**

---

## Field reference

| Field | Meaning |
|---|---|
| `schema` | Manifest format version. Currently `1`. **If an app sees a `schema` it does not know, it must do nothing** — not guess, not partially parse. That is what lets us change this file later without breaking older apps. |
| `product` | **Must be matched exactly.** `pilotADSBos`. See the fail-closed rule below. |
| `generated` | When this manifest was last written (UTC, RFC 3339). |
| `tier2.version` | SemVer of the newest full image. |
| `tier2.filename` | Exact filename the user should end up with. Worth showing — Tim asked for versioned filenames specifically so a glance confirms the right build. |
| `tier2.url` | Direct download of the `.img.xz`. |
| `tier2.sha256` | Lowercase hex of the `.img.xz` **as downloaded** (compressed — not of the `.img` inside). |
| `tier2.size` | Exact byte length. Show it before downloading. |
| `tier2.released` | Publication timestamp (UTC, RFC 3339). |
| `tier2.release_page` | Human-readable GitHub release page with the notes. |
| `tier2.instructions_url` | Flashing instructions for a non-technical user. |
| `tier2.notes` | One or two sentences, written for the end user, not for us. |

`min_app_version` is **not** present. Absent means "no minimum". CC#1 owns
whether we need one — adding it later is backward compatible, and guessing a
value now could lock users out of an update they can perfectly well install.

---

## The fail-closed product rule (this has bitten us once already)

The update check is keyed on `product` and is **fail-closed**: if the string
does not match exactly, the app must offer **nothing**.

This exists so pilotADSBos is never offered to a unit running Stratux or r21
firmware — flashing our image onto third-party hardware on the strength of a
loose version comparison would be a genuinely destructive outcome.

The cost of fail-closed is that a mismatch is **silent**. On 2026-07-19 the OS
product key changed from `pilotADSBos-UAS` to `pilotADSBos`; until the app is
re-keyed to match, units are simply never offered an update and **nothing is
logged anywhere**. If updates appear to "not work", check this string first.

- OS emits `product` in the FFE4 `ver` string, on the :63094 presence beacon,
  and at `/dp_adsb_version.json`.
- r21 self-identifies as `stratux-r21` and is unaffected.

---

## App behaviour (from `docs/CC2_OTA_APP_REQUIREMENTS_FOR_CC1.md`)

- **Never auto-download.** The image is ~566 MB. Requires an explicit, informed
  tap, and warn plainly if the user is on cellular.
- **Word tier 2 differently from tier 1.** Tier 2 needs a computer, a card
  reader and ten minutes — not a phone tap. A user who expects a quick in-app
  update and hits a card reflash will read it as a broken feature.
- **Never prompt in flight**, and never while the receiver is the sole
  navigation source. Pre-flight or post-flight only.
- **Compare against the unit's real version**, read from the receiver, not
  against a cached value.
- Show `sha256` and `size` so a user can verify what they downloaded.

---

## Releasing a new version — the checklist

1. Publish the GitHub release with the `.img.xz` and its `.sha256`.
2. Update `tier2` here: `version`, `filename`, `url`, `sha256`, `size`,
   `released`, `release_page`, `notes`.
3. Update `generated`.
4. **Take `sha256` and `size` from the published asset, not from the build log.**
   The number that matters is what a user actually downloads. Verify with:

```bash
curl -sL https://api.github.com/repos/3dMappingTools/pilotADSBos-releases/releases/latest | grep -E '"(name|size)"'
```

5. Commit and push. Allow ~5 minutes for the raw CDN.
6. Update the download button's version and size on the BYOD web page — it is
   maintained separately and will otherwise drift.
