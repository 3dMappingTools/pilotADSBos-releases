# `latest.json` — the update manifest

**Fetch URL (stable, never changes):**

```
https://updates.pilotadsb.com/latest.json
```

**Use this — not `raw.githubusercontent.com`, not a release-asset URL.** The fetch URL is baked
into every shipped app, so it must be one *we* control forever. Today it is a CNAME to GitHub
Pages; if we outgrow Pages, move hosting, or GitHub changes its terms, that becomes a single DNS
change instead of a store update users may never install.

Release-asset URLs are worse still: they contain the tag, so an app pinned to one stops seeing
updates the moment the next version ships.

HTTPS is enforced and plain HTTP 301-redirects. The content is `latest.json` at the root of this
repo, so publishing an update is an ordinary commit. Allow a few minutes for CDN propagation —
fine for a check that runs at most once a day.

---

## What is published today

Both tiers, both on **0.24.23**.

- **tier1** — a signed `dpcore` binary the app pushes to the receiver. Seconds, no card removal.
- **tier2** — the full `.img.xz`, which needs a computer and a card reader.

A unit that qualifies for tier 1 (see `min_ota_from`) should be offered **tier 1 only**. Tier 2
exists for everything else.

**Apps must treat a missing `tier1` key as "no tier-1 update available" — never as an error, and
never as a reason to fall back to tier 2.**

---

## Field reference

| Field | Meaning |
|---|---|
| `schema` | Manifest format version. Currently `1`. **If an app sees a `schema` it does not know, it must do nothing** — not guess, not partially parse. That is what lets us change this file later without breaking older apps. |
| `product` | **Must be matched exactly: `pilotADSBos`** (capital ADSB, lowercase os). See the fail-closed rule below. |
| `generated` | When this manifest was last written (UTC, RFC 3339). |
| `severity` | **Optional.** `"critical"` or `"recommended"`. **OMIT ENTIRELY for ordinary releases** — an absent value means "recommended", so a routine build needs no field. Present at the top level or inside `tier1`; the app reads either. |

### `severity` — styling only, never timing

Tim's ruling: *"If an update was considered 'Critical' then you would include that in the update
notices (both locations)"* — the startup panel and the Receiver Settings row.

**It changes STYLING ONLY. It must never shorten or override the user's 1–24 h snooze.** Tim again:
*"Their safety in using the product is more important than an update."* A receiver mid-flight is
doing its job; an update notice is not more important than that.

Set it deliberately and rarely. A release flagged critical every time teaches people to ignore the
flag, which costs us the one time it matters.


### `tier1` — over-the-air `dpcore` replacement

| Field | Meaning |
|---|---|
| `version` | SemVer of the payload. |
| `arch` | GOARCH the binary was built for. **Signed AND checked** by the receiver — a valid `arm` payload on an `arm64` unit is refused outright. Read the unit's own `arch` from `GET /update/status`; do not assume. |
| `min_ota_from` | Oldest receiver version that may take the binary-only path. **Older than this must be offered tier 2 instead.** Absent means the app should treat every unit as tier 2. |
| `url` | Direct download of the payload. |
| `sha256` / `size` | Of the payload bytes. The receiver checks these **separately from the signature**, deliberately: an interrupted download then reads as *"retry"* rather than *"someone tampered with your firmware"*. Surface those two cases differently. |
| `signature` | Ed25519 over a domain-tagged (version, arch, sha256, size) manifest, base64url unpadded. Pass through verbatim. |
| `notes` | One or two sentences for the end user. |

### `tier2` — full image, reflash required

| Field | Meaning |
|---|---|
| `version` | SemVer of the newest full image. |
| `filename` | Exact filename the user should end up with. Versioned deliberately, so a glance confirms the right build. |
| `url` | Direct download of the `.img.xz`. **Stays on GitHub Releases**, not Pages — the image is ~568 MB and Pages has a 100 GB/month soft bandwidth limit that DIY downloads would eat. |
| `sha256` | Of the `.img.xz` **as downloaded** (compressed — not of the `.img` inside). |
| `size` | Exact byte length. Show it before downloading. |
| `released` / `release_page` / `instructions_url` / `notes` | Publication time, human-readable notes page, flashing instructions, end-user summary. |

`min_app_version` is **not** present. Absent means "no minimum" — guessing a value could lock
users out of an update they can perfectly well install.

---

## The fail-closed product rule (this has bitten us once already)

The update check is keyed on `product` and is **fail-closed**: if the string does not match
exactly, the app must offer **nothing**.

This exists so pilotADSBos is never offered to a unit running Stratux or r21 firmware — flashing
our image onto third-party hardware on the strength of a loose version comparison would be
genuinely destructive.

The cost is that a mismatch is **silent**. On 2026-07-19 the OS product key changed from
`pilotADSBos-UAS` to `pilotADSBos`; until an app is re-keyed to match, units are simply never
offered an update and **nothing is logged anywhere**.

⚠️ **The published string is `pilotADSBos`.** A case-sensitive comparison against `pilotadsbos`
fails closed on every unit in the field. If updates appear to "not work", check this string first.

- OS emits `product` in the FFE4 `ver` string, on the `:63094` presence beacon, and at
  `/dp_adsb_version.json`.
- r21 self-identifies as `stratux-r21` and is unaffected.

---

## App behaviour

- **Never auto-apply.** User-initiated, on the ground, never in flight, and never while the
  receiver is the sole navigation source.
- **Never auto-download tier 2.** ~568 MB. Explicit informed tap, and warn plainly on cellular.
- **Word tier 2 differently from tier 1.** Tier 2 needs a computer, a card reader and ten
  minutes. A user expecting a quick in-app update who hits a card reflash reads it as broken.
- **Compare against the unit's real version**, read from the receiver, not a cached value.
- **Confirm a tier-1 update by re-reading the version, not by the HTTP 200.** A binary that fails
  to start three times is rolled back automatically; probation clears only after 90 s of healthy
  running. "Reconnected, version unchanged" means it reverted safely — say so plainly.
- Do not retry a **403** automatically.

Measured on real hardware 2026-07-29 — full cold boot to BLE-ready: **Pi 4B ≤ 15 s, Pi 3B ≤ 16 s**.
An OTA restarts the service only, so the real figure is lower. A 90 s confirm window is ample.

---

## Releasing a new version — the checklist

1. Publish the GitHub release with the `.img.xz` and its `.sha256`.
2. **Build `dpcore` on the build VM at the same commit as the image**, so an OTA'd unit and a
   freshly flashed card run identical bytes. Verify the version string *inside* the binary — a
   stale root-owned file once claimed a version it did not contain.
3. Sign it with `cmd/sign-update` **on the machine holding the key** (the key never travels).
4. Copy the payload into `ota/<version>/` here.
5. Update `tier1` (version, url, sha256, size, signature, `min_ota_from`) and `tier2` (version,
   filename, url, sha256, size, released, release_page, notes), then `generated`.
6. **Take `sha256` and `size` from the PUBLISHED artefacts, not the build log** — what a user
   actually downloads is the only number worth publishing. Verify by fetching and re-hashing:

```bash
curl -sL https://updates.pilotadsb.com/latest.json
```

7. Commit and push, then confirm both URLs serve and their sizes match the manifest.
8. Update the download button's version and size on the BYOD web page — maintained separately,
   and it will otherwise drift.
