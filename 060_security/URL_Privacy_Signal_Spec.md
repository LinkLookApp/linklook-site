# URL Privacy Signal Specification

> **Status:** Draft — v2
> **Date:** 2026-03-17
> **Scope:** On-device detection of sensitive data in URLs, user-controlled
> privacy dialog before any URL leaves the device for backend analysis.
> **Enforceable rules:** `CLAUDE.md` (rules 89–99). This document describes
> intent, constraints, rationale, UX copy, architecture, and rollout roadmap.

---

## 1. Purpose

URLs frequently contain personal or sensitive data — email addresses, account
IDs, tokens, reset codes, names, and more. Before LinkLook sends any URL to
the backend (Cloud URL Check, Safe Preview, or Deep Page Analysis), it must
detect these items on-device and give the user transparent control over what
is transmitted.

This is not positioned as a legal consent screen. It is positioned as a
**trust, transparency, and control** screen — a quiet privacy safeguard.

Under GDPR Article 25, data protection by design and by default must be built
into processing from the start, including data minimisation and user safeguards.
This feature fulfills that obligation at the earliest possible moment: before
any URL data leaves the device.

## 2. Core Principle

> LinkLook detects sensitive data in URLs **before** they leave the device.
> The user decides how to proceed: privacy-protected check, full check, or
> cancel. LinkLook never silently strips or sends sensitive URL data.

---

## 3. When It Triggers

The Privacy Signal activates when **all** of the following are true:

1. The URL is about to be sent to a backend endpoint (`/analyze-url`,
   `/fetch-preview`, or `/analyze-page`).
2. The on-device sensitive-data detector finds one or more matches in the URL
   (query parameters, path segments, fragment).
3. There is no permanent dismissal — the Privacy Signal **always** appears
   when sensitive data is detected (`CLAUDE.md` rule 98).

The Privacy Signal does **not** trigger for:

- On-device-only checks (structural analysis, GSB hash-prefix lookup, on-device
  AI URL analysis). These never send the full URL to LinkLook's server.
- URLs with no detected sensitive data.

---

## 4. Detected Data Types

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
- Treat these as **high-risk by default**: reset links, magic links, auth
  tokens, session identifiers, anything JWT-like, credentials or account-access
  values in query strings.
- Never log the raw URL before the privacy scan finishes. The whole point of
  client-side scanning is lost if crash logs, analytics, or debug logs already
  captured the sensitive query string.

---

## 5. UX Flow

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

The dialog is presented as a sheet. See §6 for the recommended copy and all
copy options.

```
┌──────────────────────────────────────┐
│                                      │
│    🔒 Privacy protection available   │
│                                      │
│  This link appears to include        │
│  personal or sensitive information.  │
│  To protect your privacy, LinkLook   │
│  limits what is sent for checking    │
│  whenever possible.                  │
│                                      │
│  Detected in this link               │
│  ┌──────────────────────────────────┐│
│  │ • Email address                  ││
│  │ • Account or customer ID        ││
│  │ • Security token                ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │  ✅ Continue with privacy        ││
│  │     protection (recommended)     ││
│  └──────────────────────────────────┘│
│  ┌──────────────────────────────────┐│
│  │  Use full link for deeper check  ││
│  └──────────────────────────────────┘│
│  ┌──────────────────────────────────┐│
│  │  Cancel                          ││
│  └──────────────────────────────────┘│
│                                      │
│  Show masked details                 │
│                                      │
└──────────────────────────────────────┘
```

- **Primary action** (green fill, prominent): sends masked URL.
- **Secondary action** (outlined): sends full URL. Subtext:
  "The full link will be sent to LinkLook's server."
- **Cancel**: returns to verdict screen, no backend call.
- **"Show masked details"**: tappable disclosure (see Step 3).

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

Alternative footer wording: "For your privacy, detected values are partially
hidden."

### Step 4 — Choice applies to session

If the user chooses "Privacy-protected check" for Cloud URL Check, the same
masked URL is used for any subsequent Safe Preview or Deep Page Analysis in
that check session, unless the user explicitly upgrades to "Full check."

### Step 5 — Backend receives masked or full URL

See §9 for backend requirements.

---

## 6. UI Copy Catalog

> **Single source of truth for all Privacy Signal copy.** This section
> contains every copy variant considered, the recommended starting version,
> and the premium variant. Code should implement the recommended version
> (§6.5) for Free tier and the premium variant (§6.7) for Pro tier.

### 6.1 Copy Option A — Balanced and friendly

**Title:** This link may contain sensitive data

**Body:** We found information in this link that may relate to you or another
person. For your privacy, LinkLook limits what is sent for checking whenever
possible.

**List header:** Detected data types

**How LinkLook protects (expandable):**
- Masks sensitive values where possible
- Sends only the minimum data needed for checking
- Avoids storing full values longer than necessary

**Actions:** Check with privacy protection / Send full link for deeper check / Cancel

**Disclosure:** Show detected details

### 6.2 Copy Option B — Premium and reassuring

**Title:** Privacy protection active

**Body:** This link appears to contain personal or sensitive data. LinkLook
has detected this before sending the link for analysis.

**List header:** Potentially sensitive data found

**Default protection line:** LinkLook will mask sensitive parts and send only
the minimum needed for checking.

**Actions:** Continue with privacy protection / Full analysis / Back

**Disclosure:** See what was detected

### 6.3 Copy Option C — Most explicit and plain

**Title:** Sensitive data found in this link

**Body:** This link may contain personal information. LinkLook can check the
link in a privacy-protected way, or you can choose a full check.

**List header:** Detected items

**Actions:** Privacy-protected check / Full check / Cancel

**Disclosure:** View masked details

### 6.4 Apple-like alternative — Shorter, cleaner

**Title:** Privacy protection available

**Body:** This link may include personal information. LinkLook can continue
with privacy protection, or you can choose a full check.

**List header:** Detected in this link

**Actions:** Continue Protected / Full Check / Cancel

**Disclosure:** Show masked details

This version is shorter, cleaner, and more understated. It feels closer to a
system-style privacy message while still sounding like LinkLook.

### 6.5 Recommended starting version (Free tier) ★

> **Use this version for v1 launch.**

**Title:** Privacy protection available

**Body:** This link appears to include personal or sensitive information. To
protect your privacy, LinkLook limits what is sent for checking whenever
possible.

**List header:** Detected in this link

**How LinkLook protects your privacy (expandable):**
- Masks sensitive values where possible
- Sends only the minimum data needed for checking
- Limits use of full values to what is necessary

**Actions:**
- **Continue with privacy protection** (primary, green fill, recommended)
- **Use full link for deeper check** (secondary, outlined)
- **Cancel** (tertiary)

**Disclosure:** Show masked details

**Masked footer:** Sensitive values are partially hidden on this screen.

### 6.6 Short version (for testing / compact layouts)

**Title:** Sensitive data found

**Body:** This link may include personal information.

**List:** Email address / ID / Token

**Actions:** Protected check / Full check / Cancel

**Disclosure:** View masked details

### 6.7 Premium variant (Pro tier) ★

**Title:** Your privacy is being protected

**Body:** This link appears to contain personal or sensitive information.
LinkLook has detected this before sending the link for analysis.

**List header:** Potentially sensitive data found

**Actions:**
- **Continue with Protection** (primary, green fill)
- **Use Full Link** (secondary, outlined)
- **Cancel** (tertiary)

**Disclosure:** Show masked details

Functional behavior is identical to Free. The copy signals premium care.

---

## 7. Button Microcopy Reference

### Primary action (safer / recommended)

| Option | Copy |
|--------|------|
| Recommended (Free) | Continue with privacy protection |
| Apple-like | Continue Protected |
| Explicit | Privacy-protected check |
| Short | Protected check |
| Premium (Pro) | Continue with Protection |

### Secondary action (more permissive)

| Option | Copy |
|--------|------|
| Recommended (Free) | Use full link for deeper check |
| Apple-like | Full Check |
| Explicit | Full check |
| Short | Full check |
| Premium (Pro) | Use Full Link |

### Exit action

| Option | Copy |
|--------|------|
| Recommended | Cancel |
| Alternative | Back |
| Alternative | Go back |

---

## 8. Tone Guidelines

The tone should feel:

- **Calm** — no urgency, no alarm
- **Clear** — the user understands what is happening
- **Protective** — LinkLook is on the user's side
- **Respectful** — the user is in control
- **Understated** — nothing dramatic or overwrought
- **Premium** — quality language, not discount-bin copy

Avoid:

- Alarmist language ("Warning!", "Danger!")
- Heavy legal wording ("By proceeding you consent to…")
- Defensive wording ("We are not responsible for…")
- Vague claims ("100% private", "completely anonymous")
- Technical jargon unless behind a secondary disclosure

Mental model: not a warning siren, not a legal notice — **a quiet privacy
safeguard**.

---

## 9. Masking Rules

Masking shows enough context for the user to recognize what was detected,
but never reveals the full value.

| Data type | Masking rule | Example |
|-----------|-------------|---------|
| Email | First 2 chars + `***` + `@` + domain | `ba***@example.com` |
| Phone | `***` + last 4 digits | `***5678` |
| Account/customer ID | `***` + last 4 digits | `***4831` |
| Token / reset code | 12 bullet characters | `••••••••••••` |
| Name | First 2 chars + `***` | `Ba***` |
| Address/location | First 3 chars + `***` | `101***` |
| Health terms | Shown as-is (keywords, not personal values) | `diabetes` |

Masking preserves URL structure: scheme, host, path structure, and query
parameter names are never masked. Only detected values are replaced.

---

## 10. Privacy-Protected Check: What Happens

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

---

## 11. Interaction with Cloud Analysis Consent Model and Two-Step Analysis

### Consent model

The Privacy Signal is a **separate, complementary layer** to the Cloud Analysis
Consent toggles (spec: `Cloud_Analysis_Consent_Spec.md`).

- **Consent toggles** control *whether* a URL goes to the backend at all.
- **Privacy Signal** controls *how* a URL goes to the backend — masked or full.

### Two-step analysis model

> Full spec: `LinkLook-Docs/050_architecture/URL_Webpage_Analysis_Model.md`

URL checking and webpage checking are **two separate steps**. The Privacy
Signal produces the **stripped URL** (artifact 3.2 in the analysis model),
which is the privacy-reduced URL used for display and backend analysis. This
same masking concept extends to the webpage phase: the backend generates a
**stripped webpage representation** (artifact 3.4) for AI analysis.

The Privacy Signal applies at both steps:
- **URL phase:** Masked URL is sent for Cloud URL Check.
- **Webpage phase:** The same masked or full URL (per user choice) is used to
  fetch the page. The backend then generates a stripped webpage representation
  for AI analysis (privacy-reduced by default).

### Decision ladder (two-step)

**Step 1 — URL Phase:**
1. On-device analysis (always runs first).
2. Sensitive data detection (on-device, before any backend call).
3. If sensitive data found → show Privacy Signal → user chooses masked/full/cancel.
4. Cloud URL analysis (if `url_analysis` scope is granted, uses masked or full
   URL per user choice).

**Step 2 — Webpage Phase (explicit user action only):**
5. Safe Preview (if `page_fetch` scope is granted AND user taps "Preview page").
6. Deep Page Analysis (if `page_ai_analysis` scope is granted AND user taps
   "Check webpage with AI"). Privacy-reduced by default; full mode only by
   explicit escalation.

The Privacy Signal choice persists across both steps. If the user chose
"Privacy-protected check" for Cloud URL Check, the same masked URL is used
for any subsequent Safe Preview or Deep Page Analysis request in that session
— unless the user explicitly upgrades to "Full check."

---

## 12. Architecture: Hybrid Engine + Rules File

> **Architecture decision:** Stable detection logic lives in Swift code.
> Changeable detection criteria live in a bundled JSON rules file. This is
> the right balance for v1: easy to tune rules without touching engine logic,
> easy to review rules in isolation, easy to test with different rule sets.

### What stays in code (engine)

- URL parsing with `URLComponents`
- Rule application order and pipeline
- Confidence scoring and threshold logic
- Masking functions
- Decision thresholds
- Safety guards
- Deduplication

### What moves to a bundled rules file

- Sensitive key names (e.g. `email`, `phone`, `userid`, `token`)
- Regex patterns for value detection
- Context keywords (e.g. `reset`, `invite`, `patient`)
- Per-type severity weights/scores
- UI labels (e.g. "Email address", "Security token")

### Rules file format

Bundled as `privacy_rules_v1.json` inside the app bundle or Swift Package
resource. Example shape:

```json
{
  "version": 1,
  "sensitiveKeys": {
    "email":     { "kind": "email",         "score": 4 },
    "mail":      { "kind": "email",         "score": 4 },
    "phone":     { "kind": "phone",         "score": 3 },
    "mobile":    { "kind": "phone",         "score": 3 },
    "tel":       { "kind": "phone",         "score": 3 },
    "name":      { "kind": "personName",    "score": 2 },
    "firstname": { "kind": "personName",    "score": 2 },
    "lastname":  { "kind": "personName",    "score": 2 },
    "user":      { "kind": "userId",        "score": 3 },
    "userid":    { "kind": "userId",        "score": 3 },
    "customer":  { "kind": "accountId",     "score": 3 },
    "account":   { "kind": "accountId",     "score": 3 },
    "token":     { "kind": "token",         "score": 5 },
    "auth":      { "kind": "token",         "score": 5 },
    "session":   { "kind": "token",         "score": 5 },
    "jwt":       { "kind": "jwt",           "score": 5 },
    "reset":     { "kind": "token",         "score": 5 },
    "dob":       { "kind": "dateOfBirth",   "score": 3 },
    "birthdate": { "kind": "dateOfBirth",   "score": 3 },
    "iban":      { "kind": "iban",          "score": 5 },
    "patient":   { "kind": "healthContext", "score": 5 },
    "therapy":   { "kind": "healthContext", "score": 5 },
    "oncology":  { "kind": "healthContext", "score": 5 }
  },
  "patterns": [
    {
      "name": "emailPattern",
      "kind": "email",
      "regex": "[A-Z0-9._%+-]+@[A-Z0-9.-]+\\.[A-Z]{2,}",
      "options": ["caseInsensitive"],
      "score": 4
    },
    {
      "name": "phonePattern",
      "kind": "phone",
      "regex": "\\+?[0-9][0-9\\-\\(\\) ]{6,}[0-9]",
      "options": [],
      "score": 3
    },
    {
      "name": "uuidPattern",
      "kind": "uuid",
      "regex": "\\b[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[1-5][0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}\\b",
      "options": [],
      "score": 2
    },
    {
      "name": "jwtPattern",
      "kind": "jwt",
      "regex": "^[A-Za-z0-9_-]+\\.[A-Za-z0-9_-]+\\.[A-Za-z0-9_-]+$",
      "options": [],
      "score": 5
    },
    {
      "name": "longTokenPattern",
      "kind": "token",
      "regex": "^[A-Za-z0-9\\-_~=+/]{24,}$",
      "options": [],
      "score": 4
    },
    {
      "name": "dobPattern",
      "kind": "dateOfBirth",
      "regex": "\\b(19|20)\\d{2}[-/](0[1-9]|1[0-2])[-/](0[1-9]|[12]\\d|3[01])\\b",
      "options": [],
      "score": 3
    }
  ],
  "contextKeywords": [
    { "word": "reset",     "kind": "token",         "score": 2 },
    { "word": "invite",    "kind": "token",         "score": 2 },
    { "word": "patient",   "kind": "healthContext", "score": 5 },
    { "word": "oncology",  "kind": "healthContext", "score": 5 },
    { "word": "therapy",   "kind": "healthContext", "score": 5 },
    { "word": "pregnancy", "kind": "healthContext", "score": 5 },
    { "word": "login",     "kind": "token",         "score": 2 },
    { "word": "password",  "kind": "token",         "score": 5 },
    { "word": "banking",   "kind": "accountId",     "score": 3 },
    { "word": "claim",     "kind": "accountId",     "score": 3 },
    { "word": "policy",    "kind": "accountId",     "score": 3 }
  ],
  "thresholds": {
    "showSignal": 5,
    "autoShowDialog": 10
  }
}
```

### Why not fully hardcoded

Fully hardcoded rules work for a prototype but become painful when you need to
tune false positives, add new key names, adjust weights, localize labels,
review rules without touching logic, or test different rule sets. The engine
changes slowly; the criteria change more often.

### Why not server-managed for v1

A remote rules system adds complexity and trust questions: versioning,
integrity checking, rollback, bad rule pushes, and offline behavior. For v1,
a local bundled file is safer and simpler. Remote rule updates can be added
later as a signed, versioned mechanism.

---

## 13. Data Model

```swift
// MARK: - Privacy detection types

enum PrivacyKind: String, Codable, CaseIterable {
    case email
    case phone
    case personName
    case userId
    case accountId
    case token
    case jwt
    case uuid
    case dateOfBirth
    case iban
    case healthContext
    case unknownSensitive
}

struct PrivacyFinding: Codable {
    let kind: PrivacyKind
    let location: String       // e.g. "query:email", "path[2]", "host"
    let original: String       // never shown to user, never logged
    let masked: String         // safe to display
    let score: Int
}

struct PrivacyScanResult: Codable {
    let findings: [PrivacyFinding]
    let totalScore: Int
    let shouldWarnUser: Bool   // totalScore >= threshold
    let redactedURL: String    // safe to send to backend
}

enum PrivacyCheckChoice {
    case protectedCheck   // send masked URL
    case fullCheck        // send original URL
    case cancel           // don't send anything
}
```

### What the scanner inspects

| URL part | What to look for |
|----------|-----------------|
| Host / subdomain | Names embedded in subdomains (e.g. `john-smith.customer.example.com`) |
| Path segments | User identifiers, names, tokens in path (e.g. `/users/bas/orders/12345`) |
| Query key names | Sensitive parameter names (e.g. `email`, `token`, `userid`) |
| Query values | Email patterns, phone numbers, long tokens, JWTs, UUIDs, dates |
| Fragment | Locally useful to inspect even if backend would not receive it |

### Scanner output

Instead of only a yes/no, the scanner returns:
- A risk score (sum of individual findings)
- A list of detected types with locations
- A masked URL for display
- A redacted URL for backend analysis

---

## 14. Implementation Notes

### On-device detector (`URLPrivacyScanner`)

- Runs synchronously during the check pipeline, after URL parsing, before any
  backend call.
- Uses three detection layers: (1) sensitive key names, (2) value patterns
  via regex, (3) context keywords.
- Returns `PrivacyScanResult` — empty findings array means no sensitive data.
- Must be fast (< 50ms). Uses compiled `NSRegularExpression` patterns.
- Always parse first with `URLComponents`, then inspect parts. URL parsers
  avoid brittle string-splitting logic.
- Lives in `LinkLookCore` so it's available to the app, Share Sheet extension,
  and Safari extension.

### Rules loader (`PrivacyRuleStore`)

- Loads `privacy_rules_v1.json` from the app bundle at launch.
- Validates the rules file structure. Falls back to a hardcoded minimum rule
  set if the file is missing or corrupt.
- Compiles regex patterns once at load time, caches compiled patterns.
- Provides the rule set to `URLPrivacyScanner`.

### URL masking engine (`URLPrivacyMasker`)

- Takes the original URL + `[PrivacyFinding]` → returns a masked URL string.
- Preserves URL structure (scheme, host, path structure, query parameter names).
- Replaces only the detected values, not parameter names.
- Replaces longest matches first to avoid partial-replacement collisions.
- Lives in `LinkLookCore`.

### Backend awareness

- The backend must accept masked URLs gracefully. A masked URL may have
  `REDACTED` values in query parameters — the backend must not error on these.
- The backend logs whether a URL was masked (boolean flag), but never logs or
  stores the original sensitive values.
- Add a `masked: true/false` field to the JWT payload or request body for
  `/analyze-url`, `/fetch-preview`, and `/analyze-page`.

### What to send to the backend

For most checks, send only:
- `host`
- `path keywords` (structural, not personal values)
- `query key names` (without values)
- Boolean flags: `hasEmail`, `hasToken`, `hasLongId`
- The `redactedURL`

That gives the backend and AI enough signal without always sending raw
sensitive values.

### Critical: no logging before masking

Never log the raw URL before the privacy scan finishes. Crash logs, analytics,
and debug logs must not capture the sensitive query string before masking.

---

## 15. Accessibility

- The privacy dialog supports Dynamic Type.
- VoiceOver reads the detected items list and button labels clearly.
- The "Show masked details" disclosure is keyboard-accessible.
- Colors meet WCAG AA contrast on both light and dark backgrounds.

---

## 16. Testing

- Unit tests for `URLPrivacyScanner`: one test per data type, plus edge cases
  (URL-encoded values, multiple sensitive items, no sensitive data, path-based
  sensitive data, fragment-based data).
- Unit tests for `PrivacyRuleStore`: loading, validation, fallback on corrupt
  file, compiled regex caching.
- Unit tests for `URLPrivacyMasker`: verify masking rules, round-trip URL
  validity, structure preservation, longest-match-first ordering.
- UI tests: verify inline signal appears when sensitive data is present,
  dialog opens on tap, each button routes correctly, masked details expand.
- Integration test: verify masked URL reaches backend, backend processes it
  without error, `masked: true` flag is present.
- Add test fixtures to `LinkLook-Docs/100_testing/` for privacy signal scenarios.

---

## 17. Privacy Policy Updates

Add to privacy policy section:

> **Sensitive Data Detection.** LinkLook detects personal or sensitive
> information in URLs before sending them for cloud analysis. When detected,
> you are given the choice to send a privacy-protected (masked) version of the
> URL or the full URL. If you choose privacy protection, sensitive values are
> replaced with placeholders before leaving your device. LinkLook never
> silently sends detected sensitive data to the server.

---

## 18. Rollout Roadmap

### V1 — Must-have privacy foundation (launch)

Everything needed for GDPR Article 25 compliance and user trust:

- Client-side `URLPrivacyScanner` with three detection layers
- Early masking/redaction via `URLPrivacyMasker` before any backend call
- No raw URL logging by default before masking completes
- Visible inline privacy signal on INFORM/WARN verdict screens
- Privacy dialog with three choices (protected / full / cancel)
- "Show masked details" disclosure
- Bundled local `privacy_rules_v1.json` rules file
- Recommended copy (§6.5) for Free tier
- Premium copy variant (§6.7) for Pro tier
- Unit and UI tests for scanner, masker, and dialog flow

### V1.1 — Operational hardening (post-launch update)

- False positive tuning based on real-world URL telemetry
- Rules file version bump (`privacy_rules_v2.json`) with refined weights
- Expanded context keyword list based on observed URL patterns
- Per-type detection accuracy metrics (precision/recall dashboard)
- Localized UI labels (Dutch, German, French as priority)
- Accessibility audit pass (Dynamic Type, VoiceOver, contrast)

### Later — Advanced privacy maturity (V2+)

- Signed remote rules updates (versioned, integrity-checked, with rollback)
- ML-based privacy detection for edge cases rule-based logic misses
- Exhaustive special-category classification (GDPR Article 9)
- Separate privacy settings area with detection history (anonymized)
- Per-domain sensitivity profile (financial sites get stricter defaults)
- Content Blocker integration: strip known tracking parameters automatically

### What is explicitly NOT in V1

- Remote rules updates (adds complexity, trust, and offline concerns)
- ML-based privacy detection (rules + regex is sufficient for launch)
- Exhaustive special-category classification (conservative detection is enough)
- A large separate privacy settings area (the dialog itself is the control)

---

## 19. Changelog

| Date | Version | Change |
|------|---------|--------|
| 2026-03-17 | v1 | Initial draft |
| 2026-03-17 | v2 | Added full UI copy catalog (§6), button microcopy reference (§7), tone guidelines (§8), hybrid architecture decision (§12), expanded data model (§13), rollout roadmap (§18), backend payload guidance (§14) |
