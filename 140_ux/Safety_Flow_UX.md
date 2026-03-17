# Safety Flow UX

> **Document purpose:** Intent, constraints, and rationale for LinkLook's
> safety flow. Enforceable rules with exact values live in `CLAUDE.md`.
> When syncing, `CLAUDE.md` wins.

## Core Principle

LinkLook is a safety-driven browser. Every URL entered or tapped is checked
before opening. The safety flow is designed to be **Apple-compliant** —
HTTP/HTTPS links always have a path to their destination. Safe Browsing
warnings are explicitly allowed by Apple's App Review Guidelines.

The flow is **proportional to risk**: safe links open instantly with a brief
confirmation toast, mildly suspicious links show a light interstitial, risky
links show a strong warning, and confirmed threats are blocked outright.

### Apple Compliance

Apple's App Review Guidelines require that browsers navigate HTTP/HTTPS links
to their requested destination. Safe Browsing interstitials (like those in
Safari and Chrome) are explicitly permitted. LinkLook's safety flow follows
this model:

- **OK** → Direct open. No blocking interstitial. Brief non-blocking toast.
- **INFORM / WARN** → Safe Browsing interstitial with a forward path ("Continue" / "Continue Anyway").
- **BLOCK** → Hard block for confirmed malicious content (equivalent to Safari/Chrome Safe Browsing blocks).
- **Search** → Direct results. No safety screen.

### Primary browser model

LinkLook keeps users in at all times. There is no "Open in other browser" button
anywhere in the app — not on verdict screens, not in the toolbar, not in menus.
The reasoning: if a user opens a link in Safari and gets scammed, the blame lands
on LinkLook ("I thought I was protected"). The only indirect escape is "Copy link"
in the share menu — high enough friction that nobody does it by accident.

For edge cases where WKWebView doesn't work (DigiD, iDEAL payments, OAuth),
LinkLook detects these flows automatically and hands off to the system silently.
This is not a user-facing option — it is invisible smart handoff.

### Familiar iPhone browser patterns

LinkLook should use familiar iPhone browser patterns so Safari users feel at
home immediately: recognizable navigation, familiar toolbar placement, and
standard gestures such as swipe-to-go-back, pull-to-refresh, and long-press
context menus. The safety check is the differentiator, not copycat branding.
LinkLook should remain clearly identifiable as LinkLook.

---

## Input Classification

The user provides input via the omnibox. LinkLook determines what it is:

### If it's a URL
- Has a scheme (`http://`, `https://`), or
- Looks like a domain (contains dots, TLD pattern)

→ Run the safety check engine → Show checking spinner → Route by verdict.

### If it's a search query
- No scheme, no dots, or clearly a search phrase (multiple words, question format)

→ Open search results in LinkLook's built-in browser. **No safety screen.**

### Entry points (priority order)
All entry points feed into the same flow. The experience after entry is identical.

| Priority | Entry Point           | Description                                                  | Release    |
|----------|-----------------------|--------------------------------------------------------------|------------|
| 1        | Omnibox (type/paste)  | Core flow. Type or paste a URL into the omnibox.             | v1.0       |
| 2        | QR code scanner       | Built-in scanner. Decoded URL enters the same flow.          | v1.x       |
| 3        | Share Sheet           | iOS Share Sheet: "Check with LinkLook" from any app.         | v1.x       |
| 4        | Screenshot analysis   | Import image → AI extracts links and text → enters flow.     | v2.0 (Pro) |
| 5        | Voice input           | Microphone on omnibox → iOS speech recognition → enters flow.| v2.0       |

---

## Verdict Screens

After the safety check completes, the flow depends on the verdict. The key
principle is **proportional friction**: safe links get zero friction, mildly
suspicious links get a light speed bump, risky links get a strong warning.

### No-Preload Principle

LinkLook NEVER loads, fetches, or renders the destination page before the user
explicitly opens it. The verdict information comes only from:

- **Domain name** — extracted from the URL, displayed prominently
- **Full URL** — secondary, small monospace
- **Verdict + reason** — from the decision engine
- **Signal badges** — from structural analysis
- **Safe Preview** — server-side screenshot (Pro feature, user-initiated)

Page titles, favicons, and other server-provided metadata are NOT fetched before
opening, because any HTTP request to the destination leaks the user's intent and
IP address to a potentially malicious server.

---

### Verdict: OK — Direct Open + Toast

The link passed all checks. **No interstitial.** The page opens immediately
in LinkLook's browser. A brief, non-blocking toast overlay confirms the check.

```
┌──────────────────────────────────────┐
│  ┌──────────────────────────────────┐│
│  │  ✅ Looks OK — opening          ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │                                  ││
│  │       (WKWebView loading         ││
│  │        the destination)          ││
│  │                                  ││
│  └──────────────────────────────────┘│
│                                      │
└──────────────────────────────────────┘
```

- **No blocking screen.** The page loads immediately.
- **Toast overlay:** "Looks OK — opening" with a green checkmark.
- **Duration:** ~1.5 seconds, then auto-dismisses. Non-interactive.
- **No buttons** on the toast. It is purely informational.
- The toast does NOT block interaction with the loading page beneath it.
- Visual treatment: green accent, rounded pill shape, semi-transparent background.

**Why no interstitial for OK?** Apple requires browsers to navigate to requested
destinations. An interstitial for every safe link adds friction without safety
value and risks App Review rejection. The toast confirms protection is active
without blocking the user.

### Uniform Interstitial Layout (INFORM & WARN)

> INFORM and WARN use the **same progressive disclosure layout** so elderly
> users learn one screen pattern. The differences between the two are:
> color (warm yellow vs orange), headline/subtext tone, and forward-button
> friction ("Continue →" vs tiny "Continue Anyway" with extra confirmation).
> The structure is identical.

#### Default view (collapsed) — applies to both INFORM and WARN

```
┌──────────────────────────────────────┐
│                                      │
│         [icon] [headline]            │
│                                      │
│           [subtext]                  │
│                                      │
│            short.link                │
│          (domain, large)             │
│                                      │
│  ┌──────────────────────────────────┐│
│  │    🔒 Safe Preview — Pro         ││
│  │    ┌────────────────────────┐    ││
│  │    │  Grey/desaturated      │    ││
│  │    │  preview (Pro)         │    ││
│  │    └────────────────────────┘    ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌────────────────────┐              │
│  │      GO BACK       │ [forward] →  │
│  └────────────────────┘  (text link) │
│                                      │
│    ▼ Learn more about this link      │
│                                      │
└──────────────────────────────────────┘
```

#### Expanded view (after tapping "Learn more") — applies to both

```
┌──────────────────────────────────────┐
│                                      │
│         [icon] [headline]            │
│                                      │
│           [subtext]                  │
│                                      │
│            short.link                │
│   https://short.link/abc123          │
│   (full URL, small monospace)        │
│                                      │
│    [Shortened link] [New domain]     │
│          (signal badges)             │
│                                      │
│  ┌──────────────────────────────────┐│
│  │    🔒 Safe Preview — Pro         ││
│  │    ┌────────────────────────┐    ││
│  │    │  Grey/desaturated      │    ││
│  │    │  preview (Pro)         │    ││
│  │    └────────────────────────┘    ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌────────────────────┐              │
│  │      GO BACK       │ [forward] →  │
│  └────────────────────┘  (text link) │
│                                      │
│    ▲ Learn more about this link      │
│                                      │
│  ┌──────────────────────────────────┐│
│  │    Check Message Context 🔍      ││
│  └──────────────────────────────────┘│
│       (medium, outlined, Pro)        │
│                                      │
│   Share verdict with…                │
│                                      │
└──────────────────────────────────────┘
```

#### Design notes (shared layout)

- **Default view has 5 elements:** headline + subtext, domain name, Safe Preview, Go Back button, forward text link. The preview gives the user an immediate visual anchor.
- **"Go Back"** is a large filled button (left-aligned). The forward action is a small text link on the same row (right-aligned). Size asymmetry steers toward Go Back.
- **Safe Preview is user-initiated.** A "Preview page" button sits between the domain and the action buttons. Pro users tap it to request a server-side screenshot (grey/desaturated on WARN, full-color on INFORM). Free users see a locked placeholder with lock icon and "Pro" badge. The preview is never auto-fetched — the user must explicitly tap to request it. This keeps the privacy story clean and easy to explain to Apple.
- **Preview unavailable state** — If the preview server is down, times out, or returns an error, Pro users see a graceful placeholder card: light grey rounded card (same dimensions as a loaded preview), centred `photo.slash` icon in `Theme.textLight` grey, text "Preview not available right now" (caption2, grey), and a small "Retry" link. The placeholder matches the height of a loaded preview so the layout does not jump. No technical error details are shown. This does not affect the verdict or any other UX element. Full spec: `Safe_Preview.md` rule 10.
- **"Learn more about this link"** is a disclosure row (▼/▲). Tapping reveals: full URL, signal badges, Check Message Context (Pro), and Share verdict. Animated expand/collapse.
- **Domain only in collapsed view.** Full URL appears in expanded section.
- The expanded section scrolls if it exceeds screen height (important for smaller iPhones).

### Verdict: INFORM — Light Warning

Minor concerns — the user should be aware but can proceed easily.

- **Headline:** "This link may be unfamiliar"
- **Subtext:** "Continue only if you trust this site."
- **Forward button:** small "Continue →" text link (`.openInApp`).
- Visual treatment: **bright yellow/advisory** accent (#FEBC2E).

### Verdict: WARN — Strong Warning

Significant concerns — the user is strongly nudged away.

- **Headline:** "This page shows warning signs"
- **Subtext:** "Going back is recommended."
- **Forward button:** very small "Continue Anyway" text (`.openInApp`, 0.6x font, light color). Tapping requires an **extra confirmation tap** ("Are you sure? This link has warning signs.").
- Visual treatment: **orange/caution** accent (#FF9500).
- Colors, icons, and wording must **clearly distinguish WARN from INFORM**.

### Verdict: BLOCK — Hard Block

Confirmed malicious — no forward path. Equivalent to Safari/Chrome Safe Browsing blocks.

```
┌──────────────────────────────────────┐
│                                      │
│         🛑 Link Blocked             │
│                                      │
│  ┌──────────────────────────────────┐│
│  │  ⚠ "This website may be         ││
│  │    pretending to be another      ││
│  │    company."                     ││
│  └──────────────────────────────────┘│
│   (reason in red card, prominent)    │
│                                      │
│  ┌──────────────────────────────────┐│
│  │      paypa1-secure.login.xyz     ││
│  │  https://paypa1-secure.login.xyz ││
│  └──────────────────────────────────┘│
│   (domain large, full URL small)     │
│                                      │
│    [Brand lookalike]                 │
│    [Character tricks]                │
│                                      │
│  ┌──────────────────────────────────┐│
│  │           GO BACK                ││
│  └──────────────────────────────────┘│
│        (large, only button)          │
│                                      │
│    "This doesn't look right?"       │
│           ↑ report link              │
│                                      │
└──────────────────────────────────────┘
```

- **One large "Go Back" button only.** No way forward.
- Always show the **reason** for blocking in a **prominent red card**.
- No Safe Preview — confirmed malicious pages are not previewed.
- Visual treatment: **red/danger** accent.

### Verdict: UNKNOWN (Could Not Check)

Engine could not complete the check.

- Same as BLOCK layout: large "Go Back" button only, plus "Retry" option.
- Visual treatment: **gray/neutral**.

---

## URL Privacy Signal (Pre-Backend Gate)

> Full spec: `LinkLook-Docs/060_security/URL_Privacy_Signal_Spec.md`
> Enforceable rules: `CLAUDE.md` rules 89–99.

Before any URL leaves the device for backend analysis, LinkLook scans it
on-device for sensitive data (email addresses, account IDs, tokens, names,
phone numbers, etc.). If sensitive data is detected, the user sees a privacy
signal and decides how to proceed.

### Where it appears in the flow

The Privacy Signal sits between the on-device verdict and any backend call:

```
URL entered → On-device check → Verdict rendered
                                      │
                           Sensitive data found?
                           ┌──── No ────┐
                           │            │
                           ▼            ▼
                  Privacy Signal    Backend call
                  (user decides)    (if enabled)
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
          Protected   Full    Cancel
           check     check   (no send)
              │        │
              ▼        ▼
          Backend call with
          masked or full URL
```

### Inline signal (on INFORM/WARN screens)

When sensitive data is detected and a cloud check is enabled, a compact
notice appears below the signal badges on the verdict screen:

> 🔒 **Privacy notice** — This link may contain personal or sensitive data.
> LinkLook will limit what is sent for checking whenever possible.
> *Review privacy details*

Tapping "Review privacy details" opens the full privacy dialog.

### Privacy dialog

Presented as a sheet with the recommended copy:

- **Title:** "Privacy protection available"
- **Body:** "This link appears to include personal or sensitive information.
  To protect your privacy, LinkLook limits what is sent for checking whenever
  possible."
- **List header:** "Detected in this link" (e.g. Email address, Account ID,
  Security token)
- **Actions:** "Continue with privacy protection" (primary, green fill) /
  "Use full link for deeper check" (secondary, outlined) / "Cancel"
- **Disclosure:** "Show masked details" → expands to show masked values

On Pro tier, the dialog uses elevated copy: title "Your privacy is being
protected", actions "Continue with Protection" / "Use Full Link" / "Cancel".

Full copy catalog with all variants: `URL_Privacy_Signal_Spec.md` §6.

### Architecture

Detection logic lives in Swift code (`URLPrivacyScanner`). Detection criteria
(sensitive key names, regex patterns, context keywords, weights) live in a
bundled `privacy_rules_v1.json` file. This allows rule tuning without code
changes. Full architecture: `URL_Privacy_Signal_Spec.md` §12.

### Interaction with consent toggles

The Privacy Signal is complementary to the Cloud Analysis Consent toggles.
Consent toggles control *whether* a URL goes to the backend. The Privacy
Signal controls *how* — masked or full. Both must pass for a backend call
to proceed.

### Tone

Calm, clear, protective. Not a warning siren, not a legal notice — a quiet
privacy safeguard. See tone guidelines in `URL_Privacy_Signal_Spec.md` §8.

### V1 scope

The privacy scan, masking, visible signal, dialog, and bundled rules file are
all V1 requirements. Remote rule updates and ML-based detection are deferred.
Full roadmap: `URL_Privacy_Signal_Spec.md` §18.

---

## Safe Preview (Pro Feature)

Safe Preview uses a server-side proxy (`linklook-backend.fly.dev`) that renders
the destination page in a sandbox and returns a JPEG screenshot. The user sees
the page visually without loading it on their device. No JS runs on the user's
device, no cookies, no IP leak.

### Per-Verdict Behavior

| Verdict | Free User                                    | Pro User                                        |
|---------|----------------------------------------------|-------------------------------------------------|
| OK      | No preview (page opens immediately)          | No preview (page opens immediately)             |
| INFORM  | Locked placeholder: "Safe Preview — Pro" 🔒  | "Preview page" button → tap to fetch full-color preview  |
| WARN    | Locked placeholder: "Safe Preview — Pro" 🔒  | "Preview page" button → tap to fetch grey/desaturated preview |
| BLOCK   | No preview                                   | No preview                                      |

### Design Details

- **INFORM and WARN preview is user-initiated** — a "Preview page" button sits
  in both interstitials. Tapping fetches a server-side screenshot (full-color on
  INFORM, grey/desaturated on WARN). The preview gives the user a visual anchor
  ("does this look like my bank?") without loading the page on their device.
  This is especially helpful for elderly users who may not recognize a domain
  name but would recognize a fake page visually. The preview is never auto-fetched.
- **Free users see a locked placeholder** — a card with a lock icon and
  "Safe Preview — Pro" text. Tapping shows a brief upgrade prompt. This follows
  the locked-but-visible pattern: show Pro features at the moment of highest
  user intent.
- **Preview unavailable (Pro)** — If the preview server is down or the request
  fails, the preview area shows a placeholder card matching the dimensions of a
  loaded preview: `photo.slash` icon, "Preview not available right now" text,
  and a "Retry" link. No technical error details. Does not affect the verdict.
  See `Safe_Preview.md` rule 10.
- **Consent required** — GDPR opt-in. The first time a user uses Safe Preview,
  show a consent dialog explaining that the URL will be sent to LinkLook's
  preview service.

### Free vs Pro Philosophy

> **Core safety NEVER depends on Pro.** Free users get: verdict + reason + safe
> choice (Go Back / Continue) + signal badges. This is complete protection.
> Pro gives you more context before you decide — the preview shows what the page
> looks like, context analysis checks the message around the link. But the safety
> decision is fully informed at the Free tier.
>
> "You are protected either way. Pro gives you more context before you decide."

---

## Two-Step Analysis Model

> Full spec: `LinkLook-Docs/050_architecture/URL_Webpage_Analysis_Model.md`
> Enforceable rules: `CLAUDE.md` rules 104–120.

URL checking and webpage checking are **two separate steps**. LinkLook never
silently escalates from URL check to page fetch. The page is only fetched
when the user explicitly requests it.

### Step 1 — URL Phase (always)

The standard safety flow described above: enter URL → on-device checks →
cloud URL analysis (if enabled) → verdict screen. The user sees the URL
verdict, stripped URL, privacy notice, signal badges, and reasons.

### Step 2 — Webpage Phase (explicit user action only)

On INFORM and WARN verdict screens, a "Check webpage with AI" button lets the
user request deeper analysis. Tapping it triggers the backend to:

1. Fetch the page (via Urlbox/proxy)
2. Generate a preview page (JPEG screenshot — the Safe Preview)
3. Generate a stripped webpage representation (extracted text, headings, forms,
   CTAs, brand references — privacy-reduced)
4. AI analyzes the stripped webpage representation

The user then sees:
- Webpage verdict (may differ from the URL verdict)
- Preview page (visual screenshot)
- Reasons (what the AI found on the page)

#### Privacy-reduced analysis by default

The AI analyzes a **stripped webpage representation**, not the full page. This
preserves risk-relevant signals (fake login forms, brand impersonation,
urgency language, suspicious CTAs) while reducing privacy exposure. The
stripped version omits raw personal data, session tokens, tracking data, and
technical clutter.

#### Optional deeper analysis (V1)

On the webpage result screen, a secondary action is available:

> **Use deeper full-page analysis** — This uses more page content.

This is a secondary, advanced option — not the default path. Tapping it
triggers full-page analysis (audit logged). Most users will not need it.

#### Confidence-based escalation (V1.1)

A post-launch enhancement: if the privacy-reduced analysis cannot reach a
confident conclusion, LinkLook automatically offers:

> **Result uncertain** — Run deeper analysis? This uses more page content.

The user can choose "Run deeper analysis" (processes more page content, audit
logged) or "Keep current result" (stay with the reduced analysis). This
requires confidence scoring calibrated from real-world data.

### User-facing labels

| Mode | Label | Explanation |
|------|-------|-------------|
| Default | "Private check" | "Analyzes a reduced page version" |
| Deeper | "Deeper check" | "Analyzes more page content for higher accuracy" |

Do NOT use "stripped" and "full" as user-facing labels.

### "Check webpage with AI" button

- Shown on INFORM and WARN verdict screens only.
- Medium-sized button, positioned in the expanded "Learn more" section.
- Tapping shows a brief explanation: "LinkLook will fetch the page and analyze
  a privacy-reduced version."
- Pro feature (Early Access: free for all users).

### Audit logging

When the user selects deeper/full-page analysis, LinkLook records a minimal
audit event: device ID (pseudonymous), timestamp, event type, URL hash,
analysis mode chosen. The log records the decision, never the page content.

---

## Context Analysis Flow (Pro Feature)

Available on INFORM and WARN interstitials via the "Check Message Context" button.

### Purpose
URL analysis alone has limits. A fresh phishing domain with valid SSL can look
clean. But the MESSAGE around the link almost always contains social engineering
tells. Context analysis catches what URL checks miss.

### Flow

1. User taps "Check Message Context" on an INFORM or WARN screen.
2. **Privacy disclaimer** appears:
   > "The text you paste will be sent to LinkLook's AI service for analysis.
   > It may contain personal information. Only paste what you're comfortable sharing."
   - User taps "I understand" to continue or "Cancel" to go back.
3. **Text input screen** appears with a large text field.
   - Placeholder: "Paste the message that contained this link"
   - "Analyze" button (disabled until text is entered).
4. AI agent analyzes the text and scores it on:
   - **Urgency/pressure** — "Act now!", deadlines, threats
   - **Unsolicited contact** — Did you start this conversation?
   - **Authority impersonation** — Pretending to be a bank, government, Apple, etc.
   - **Emotional manipulation** — Fear, greed, curiosity
   - **Request for sensitive data** — Passwords, payment info, personal details
   - **Grammar/formatting anomalies** — Common in mass phishing campaigns
5. **Result screen** shows:
   - Overall verdict: traffic-light style (red/amber/green)
     - "This message shows strong signs of a scam"
     - "Some warning signs detected"
     - "This message looks normal"
   - 3–4 scored indicators with one-line explanations
   - Caveat: "This analysis is based on the text you provided.
     It doesn't guarantee the link is safe."
6. User returns to the interstitial with the context result visible as
   additional information. The original buttons remain unchanged.

### Data handling
- Text is processed and scored, then **immediately discarded**. Never stored.
- Must be disclosed in App Store privacy nutrition labels.

---

## Additional Features

### "Ask Someone You Trust" (INFORM / WARN screens)
On INFORM and WARN screens, a secondary "Share verdict with…" option
lets the user send the URL, verdict, and analysis summary to a trusted contact
via iMessage or WhatsApp. Not a primary button — appears as a link or icon.

### QR Code Scanner
Built-in camera-based QR code scanner. Decoded URL enters the exact same
safety flow. Accessed from a camera icon on the Home screen or omnibox.

### Share Sheet Integration
iOS Share Sheet extension: "Check with LinkLook". Available when the user
long-presses a link in any app. Feeds the URL into the check pipeline.

### Screenshot / Image Analysis (Pro)
User imports a screenshot of a suspicious message. AI extracts text and visible
links from the image, then runs both URL analysis and context analysis.

### Voice Input
Microphone button on the omnibox. Uses iOS native speech recognition.
Accessibility feature for elderly users.

---

## Free / Pro Tiers

### Phase 1: Early Access (v1.0 launch)

All features unlocked. No paywall, no StoreKit, no feature gates.
Pro features display a small **"Free during Early Access"** badge.

### Phase 2: Pro launches (post-launch update)

Early Access ends. Pro features become gated behind a subscription.
Free users see Pro features as **visible but locked** (locked-but-visible pattern).

### What is always Free

The entire core safety product and browser:

- Safety flow (enter link → check → verdict/toast → open in LinkLook)
- Structural analysis + Google Safe Browsing checks
- All verdict screens (OK toast, INFORM, WARN, BLOCK)
- All entry points (omnibox, QR, clipboard, Share Sheet)
- Full browser features: tabs, favorites, reader mode, downloads
- History and Recent tab
- Voice input
- Password AutoFill (via iOS system keychain)

### What is Pro

- **Safe Preview** — server-side page screenshot on INFORM/WARN screens
- **On-device AI URL analysis** (Core ML, third signal in the check pipeline)
- **Context analysis** ("Check Message Context" button)
- **Screenshot/image analysis**
- **"AI-enhanced protection"** label during checking spinner

### Per-screen differences (Phase 2)

| Verdict  | Free user                                                          | Pro user                                            |
|----------|---------------------------------------------------------------------|-----------------------------------------------------|
| OK       | Toast + open (structural + GSB)                                     | Toast + open (structural + GSB + AI URL)            |
| INFORM   | Interstitial; locked preview placeholder; context analysis locked 🔒 | Interstitial; "Preview page" button (tap to fetch); context analysis  |
| WARN     | Interstitial; locked preview placeholder; context analysis locked 🔒 | Interstitial; "Preview page" button (tap to fetch); context analysis  |
| BLOCK    | Identical                                                           | Identical                                           |
| UNKNOWN  | Identical                                                           | Identical                                           |
| Checking | "Basic protection"                                                  | "AI-enhanced protection"                            |

---

## Check Pipeline (Internals)

When a URL enters the pipeline, the decision engine runs checks **in parallel**:

```
URL entered
    │
    ├───────────────┬──────────────────┬─────────────────┐
    │               │                  │                 │
    ▼               ▼                  ▼                 │
┌──────────────┐ ┌────────────────┐ ┌──────────────┐    │
│  LinkLook    │ │  Google Safe   │ │  On-device   │    │
│  Structural  │ │  Browsing      │ │  AI URL      │    │
│  Analysis    │ │  (GSB)         │ │  Analysis    │    │
│              │ │                │ │  (Pro only)  │    │
│ • Domain     │ │ • Hash-prefix  │ │              │    │
│   patterns   │ │   query        │ │ • Core ML    │    │
│ • TLD check  │ │ • Threat types │ │   model      │    │
│ • IP detect  │ │ • 3s timeout   │ │ • On-device  │    │
│ • Brand      │ │                │ │   only       │    │
│   lookalike  │ │                │ │              │    │
│ • Redirect   │ │                │ │              │    │
│   chains     │ │                │ │              │    │
└──────┬───────┘ └───────┬────────┘ └──────┬───────┘    │
       │                 │                 │             │
       └────────┬────────┴─────────────────┘             │
                │                                        │
                ▼                          Free users ───┘
        ┌───────────────┐                  skip AI URL
        │ Merge signals │                  analysis
        │ Highest       │
        │ severity wins │
        │               │
        │ OK < INFORM   │
        │ < WARN < BLOCK│
        └───────┬───────┘
                │
                ▼
          Route by verdict
                │
                ▼
       ┌────────────────────┐
       │ Privacy Signal     │  ← Only if backend call needed
       │ (on-device scan    │    AND sensitive data found
       │  for sensitive     │
       │  data in URL)      │
       │                    │
       │ User chooses:      │
       │ • Protected check  │
       │ • Full check       │
       │ • Cancel           │
       └────────┬───────────┘
                │
                ▼
         Backend call
         (if enabled)
```

### No safe-URL caching

LinkLook runs a **fresh check on every URL, every time**. Never skip or
shorten a check based on a previous safe result. The only caching allowed
is GSB result caching (5 minutes) to avoid redundant API calls.

---

## Design Principles

1. **Proportional friction.** Safe links: zero friction (toast). Suspicious links:
   light speed bump. Risky links: strong warning. Confirmed threats: hard block.
   The friction matches the risk.
2. **Apple-compliant.** HTTP/HTTPS links always have a path to their destination
   (except confirmed malicious BLOCK). Interstitials follow the Safe Browsing
   pattern used by Safari and Chrome.
3. **Button hierarchy communicates risk without words.** A screen with one giant
   "Go Back" vs. one tiny "Continue Anyway" is universally understood.
4. **Colors and context differentiate verdicts.** INFORM (yellow #FEBC2E) and
   WARN (orange #FF9500) have similar layouts; visual treatment tells the story.
5. **Every entry point converges on the same flow.** QR, clipboard, share sheet,
   voice — they all feed into the same pipeline.
6. **Respect the user's ability to decide.** Even on WARN, offer a path forward
   (with extra friction). Only BLOCK removes the choice entirely.
7. **Design for the elderly audience.** Large tap targets, clear labels, no
   jargon, no ambiguity.
8. **Use familiar iPhone browser patterns.** Users should feel at home, but
   LinkLook should remain clearly identifiable. The safety check is the
   differentiator, not copycat browser chrome.
9. **Keep users in.** No exit to other browsers. Protection only works if
   LinkLook handles every link.
10. **Core safety never depends on Pro.** Free users get full verdicts, reasons,
    and safe choices. Pro adds context (preview, message analysis) — never gates
    the safety decision itself.

---

## What LinkLook Is NOT

- VPN functionality
- Its own password manager (iOS system keychain works automatically)
- Full ad/tracker blocking
- Automatic background scanning of all browsing
- An "Open in other browser" option

---

## Safari Extension: Second Defense Layer

For users still browsing in Safari, LinkLook provides a **Safari Web Extension**
and a **Content Blocker** as a second line of defense.

### What the Safari extension does

- **Search result warnings** — marks suspicious links on Google/Bing result pages
- **Landing page banners** — warning banner on pages with suspicious domains
- **"Open in LinkLook" button** — sends the URL to LinkLook for a full check
- **Domain reputation badges** — small safety indicator on every page

The Content Blocker provides hard-block rules for confirmed malicious domains.

Full spec: `CLAUDE.md` rules 62–74.
