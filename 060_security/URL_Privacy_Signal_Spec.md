# URL Privacy Signal Specification

> **Status:** Draft — v1
> **Date:** 2026-03-17
> **Scope:** On-device detection of sensitive data in URLs, user-controlled
> privacy dialog before any URL leaves the device for backend analysis.
> **Enforceable rules:** `CLAUDE.md` (rules 89–99). This document describes
> intent, constraints, rationale, and UX copy.

## Purpose

URLs frequently contain personal or sensitive data — email addresses, account
IDs, tokens, reset codes, names, and more. Before LinkLook sends any URL to
the backend (Cloud URL Check, Safe Preview, or Deep Page Analysis), it must
detect these items on-device and give the user transparent control over what
is transmitted.

This is not a legal consent screen. It is a trust, transparency, and control
screen — positioned as a quiet privacy safeguard.

## Core Principle

> LinkLook detects sensitive data in URLs **before** they leave the device.
> The user decides how to proceed: privacy-protected check, full check, or
> cancel. LinkLook never silently strips or sends sensitive URL data.

## When It Triggers

The Privacy Signal activates when **all** of the following are true:

1. The URL is about to be sent to a backend endpoint (`/analyze-url`,
   `/fetch-preview`, or `/analyze-page`).
2. The on-device sensitive-data detector finds one or more matches in the URL
   (query parameters, path segments, fragment).
3. The user has not permanently dismissed the Privacy Signal for this session
   (no "Don't ask again" — always show when sensitive data is found).

The Privacy Signal does **not** trigger for:

- On-device-only checks (structural analysis, GSB hash-prefix lookup, on-device
  AI URL analysis). These never send the full URL to LinkLook's server.
- URLs with no detected sensitive data.

## Detected Data Types

| Type | Detection method | Example match | Helper label |
|------|-----------------|---------------|-------------|
| Email address | Regex `[^@\s]+@[^@\s]+\.[a-z]{2,}` in query/path | `?email=bas@example.com` | May identify a person directly |
| Phone number | E.164 or local patterns in query/path | `?phone=+31612345678` | May identify a person directly |
| Account/customer ID | Key-value patterns (`uid=`, `cid=`, `account=`, etc.) | `?uid=482931` | May link to a personal account |
| Token / reset code | Long random strings in query/path (`token=`, `code=`, `reset=`, etc.) | `?token=a8f3…e91b` | May allow access to a private session or account |
| Name | `name=`, `user=`, `firstname=`, `lastname=` patterns | `?name=Bas+Jonkhoff` | May identify a person directly |
| Address / location | `address=`, `location=`, `lat=`, `lng=`, `zip=` patterns | `?zip=1012AB` | May reveal personal location data |
| Health-related term | Keyword list in path/query (medical terms, conditions) | `/results?condition=diabetes` | May reveal sensitive personal information |

### Detection rules

- Detection runs **on-device only**. No data leaves the device during detection.
- Detection applies to query parameters, path segments, and URL fragment.
  The domain and scheme are excluded.
- Detection is conservative: prefer false positives over missed sensitive data.
  Users can always choose "Full check" to proceed unmasked.
- The detector produces a list of `SensitiveDataItem` objects, each with:
  `type` (enum), `parameterName` (e.g. "email"), `maskedValue` (e.g.
  "ba***@example.com"), and `rawValue` (never shown to user, used only for
  masking logic).

## UX Flow

### Step 1 — Inline signal on result screen

When the on-device check completes and a verdict is rendered (INFORM or WARN),
and the URL contains sensitive data, show a compact inline notice below the
signal badges:

```
┌──────────────────────────────────────┐
│  🔒 Privacy notice                   │
│  This link may contain personal      │
│  or sensitive data.                  │
│                                      │
│  LinkLook will limit what is sent    │
│  for checking whenever possible.     │
│                                      │
│  Review privacy details              │
└──────────────────────────────────────┘
```

- Style: subtle card, `Theme.privacySignal` background (light blue-grey tint),
  lock icon, caption text.
- Position: below signal badges, above Safe Preview section.
- "Review privacy details" is a tappable link → opens the privacy dialog.

**On OK verdict:** The privacy signal does not appear (page opens immediately
via toast; no backend call unless Cloud URL Check is enabled). If Cloud URL
Check auto-runs on OK, the Privacy Signal triggers as a brief interstitial
before the backend call — same dialog, but shown inline during the check phase.

**On BLOCK verdict:** No privacy signal needed (no forward path, no preview).

### Step 2 — Privacy dialog

Triggered by:
- User taps "Review privacy details" on the inline signal.
- Automatic presentation if the URL is about to be sent to backend and
  sensitive data score is above the high-risk threshold.

```
┌──────────────────────────────────────┐
│                                      │
│    🔒 Sensitive data found           │
│       in this link                   │
│                                      │
│  This link may contain personal      │
│  information. LinkLook can check     │
│  the link in a privacy-protected     │
│  way, or you can choose a full       │
│  check.                              │
│                                      │
│  Detected items                      │
│  ┌──────────────────────────────────┐│
│  │ • Email address                  ││
│  │ • Account ID                     ││
│  │ • Token                          ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │  ✅ Privacy-protected check      ││
│  │     (recommended)                ││
│  └──────────────────────────────────┘│
│  ┌──────────────────────────────────┐│
│  │     Full check                   ││
│  └──────────────────────────────────┘│
│  ┌──────────────────────────────────┐│
│  │     Cancel                       ││
│  └──────────────────────────────────┘│
│                                      │
│  View masked details                 │
│                                      │
└──────────────────────────────────────┘
```

- **"Privacy-protected check"** is the default highlighted action (green fill,
  prominent). Sends a masked URL to the backend — detected sensitive values
  are replaced with type-preserving placeholders (e.g. `email=REDACTED@example.com`,
  `token=REDACTED`).
- **"Full check"** is secondary (outlined style). Sends the URL as-is. The
  user explicitly chooses this. Small subtext: "The full link will be sent
  to LinkLook's server."
- **"Cancel"** — returns to the verdict screen without sending anything.
  No backend call.
- **"View masked details"** — tappable disclosure link (see Step 3).

### Step 3 — Masked details disclosure

```
┌──────────────────────────────────────┐
│  Detected details                    │
│                                      │
│  Email: ba***@example.com            │
│  Account ID: ***4831                 │
│  Token: ••••••••••••                 │
│                                      │
│  Sensitive values are masked on      │
│  this screen.                        │
└──────────────────────────────────────┘
```

- Masked values show enough context for the user to recognize what was detected,
  but never reveal the full value.
- Masking rules:
  - Email: first 2 chars + `***` + `@` + domain (`ba***@example.com`)
  - IDs: `***` + last 4 digits (`***4831`)
  - Tokens: 12 bullet characters (`••••••••••••`)
  - Names: first 2 chars + `***` (`Ba***`)
  - Phone: `***` + last 4 digits (`***5678`)
  - Address/location: first 3 chars + `***` (`101***`)
  - Health terms: shown as-is (they are keywords, not personal values)

## Privacy-Protected Check: What Happens

When the user chooses "Privacy-protected check":

1. The on-device masking engine replaces each detected sensitive value in the
   URL with a type-preserving placeholder.
2. The masked URL is sent to the backend endpoint.
3. The backend analyzes the masked URL. Because structural features (domain,
   TLD, path structure, query parameter names) are preserved, the analysis
   remains effective for most threat signals.
4. The backend **never** receives the original sensitive values.

### Masking preserves analysis quality

Most phishing signals come from structural URL features, not parameter values:
domain name, TLD, path structure, number of redirects, brand lookalikes. The
masked URL retains all of these. Only the parameter values are replaced.

### When masking may reduce analysis quality

If the URL's path itself is the sensitive data (e.g. `/users/bas-jonkhoff/reset`),
masking the path segment may remove context the backend needs. In this case,
the dialog adds a note: "Some path segments may be masked, which could affect
the depth of the check."

## Premium Variant (Pro)

On Pro tier, the privacy dialog uses slightly elevated copy:

- **Title:** "Your privacy is being protected"
- **Body:** "This link appears to contain personal or sensitive information.
  LinkLook has detected this before sending the link for analysis."
- **List header:** "Potentially sensitive data found"
- **Actions:** "Continue with Protection" / "Use Full Link" / "Cancel"
- **Disclosure:** "Show masked details"

The functional behavior is identical. The copy signals premium care.

## Tone Guidelines

The tone should feel: calm, clear, protective, respectful, understated, premium.

Avoid: alarmist language, heavy legal wording, defensive wording, vague claims
like "100% private", technical jargon unless behind a secondary disclosure.

Mental model: not a warning siren, not a legal notice — a quiet privacy safeguard.

## Interaction with Cloud Analysis Consent Model

The Privacy Signal is a **separate, complementary layer** to the Cloud Analysis
Consent toggles (spec: `Cloud_Analysis_Consent_Spec.md`).

- **Consent toggles** control *whether* a URL goes to the backend at all.
- **Privacy Signal** controls *how* a URL goes to the backend — masked or full.

Decision ladder (updated):

1. On-device analysis (always runs first).
2. Sensitive data detection (on-device, before any backend call).
3. If sensitive data found → show Privacy Signal → user chooses masked/full/cancel.
4. Cloud URL analysis (if `url_analysis` scope is granted, uses masked or full
   URL per user choice).
5. Safe Preview (if `page_fetch` scope is granted AND user taps "Preview page").
6. Deep Page Analysis (if `page_ai_analysis` scope is granted).

The Privacy Signal applies to all three backend endpoints. If the user chose
"Privacy-protected check" for Cloud URL Check, the same masked URL is used
for any subsequent Safe Preview or Deep Page Analysis request in that session
— unless the user explicitly upgrades to "Full check."

## Data Model

```swift
struct SensitiveDataItem {
    let type: SensitiveDataType
    let parameterName: String      // e.g. "email", "token"
    let maskedValue: String        // e.g. "ba***@example.com"
    let rawValue: String           // never shown to user, never logged
}

enum SensitiveDataType: String, CaseIterable {
    case email
    case phone
    case accountID
    case token
    case name
    case address
    case healthTerm
}

enum PrivacyCheckChoice {
    case protectedCheck   // send masked URL
    case fullCheck        // send original URL
    case cancel           // don't send anything
}
```

## Implementation Notes

### On-device detector (`URLPrivacyScanner`)

- Runs synchronously during the check pipeline, after URL parsing, before any
  backend call.
- Returns `[SensitiveDataItem]` — empty array means no sensitive data found.
- Must be fast (< 50ms). Uses compiled regex patterns.
- Lives in `LinkLookCore` so it's available to the app, Share Sheet extension,
  and Safari extension.

### URL masking engine (`URLPrivacyMasker`)

- Takes the original URL + `[SensitiveDataItem]` → returns a masked URL string.
- Preserves URL structure (scheme, host, path, query parameter names).
- Replaces only the detected values, not parameter names.
- Lives in `LinkLookCore`.

### Backend awareness

- The backend should accept masked URLs gracefully. A masked URL may have
  `REDACTED` values in query parameters — the backend should not error on these.
- The backend should log whether a URL was masked (boolean flag), but never
  log or store the original sensitive values.
- Add a `masked: true/false` field to the JWT payload or request body for
  `/analyze-url`, `/fetch-preview`, and `/analyze-page`.

## Accessibility

- The privacy dialog supports Dynamic Type.
- VoiceOver reads the detected items list and button labels clearly.
- The "View masked details" disclosure is keyboard-accessible.
- Colors meet WCAG AA contrast on both light and dark backgrounds.

## Testing

- Unit tests for `URLPrivacyScanner`: one test per data type, plus edge cases
  (URL-encoded values, multiple sensitive items, no sensitive data, etc.).
- Unit tests for `URLPrivacyMasker`: verify masking rules, round-trip URL
  validity, structure preservation.
- UI tests: verify inline signal appears when sensitive data is present,
  dialog opens on tap, each button routes correctly.
- Integration test: verify masked URL reaches backend, backend processes it
  without error.
- Add test fixtures to `LinkLook-Docs/100_testing/` for privacy signal scenarios.

## Privacy Policy Updates

Add to privacy policy section:

> **Sensitive Data Detection.** LinkLook detects personal or sensitive
> information in URLs before sending them for cloud analysis. When detected,
> you are given the choice to send a privacy-protected (masked) version of the
> URL or the full URL. If you choose privacy protection, sensitive values are
> replaced with placeholders before leaving your device. LinkLook never
> silently sends detected sensitive data to the server.

## Changelog

| Date | Version | Change |
|------|---------|--------|
| 2026-03-17 | v1 | Initial draft |
