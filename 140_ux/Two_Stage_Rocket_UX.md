# Two-Step Safety Flow

> **Document purpose:** Intent, constraints, and rationale for the two-step safety
> flow. Enforceable rules with exact values live in `CLAUDE.md`.
> When syncing, `CLAUDE.md` wins.

## Core Principle

Every entered or tapped URL in LinkLook is handled in two steps:

1. **Check the destination.**
2. **Let the user explicitly choose what happens next.**

Users never land on a URL automatically. LinkLook always shows a verdict first.
Only after that does the user decide to go back or open the page — always inside
LinkLook. There is no option to open in another browser.

LinkLook is not "a browser that also checks links." It is the user's **primary
and only browser** — built around a safety decision point. The pause IS the
product. If users leave LinkLook, they lose protection.

### Primary browser model

LinkLook keeps users in at all times. There is no "Open in other browser" button
anywhere in the app — not on verdict screens, not in the toolbar, not in menus.
The reasoning: if a user opens a link in Safari and gets scammed, the blame lands
on LinkLook ("I thought I was protected"). The only indirect escape is "Copy link"
in the share menu — high enough friction that nobody does it by accident.

For edge cases where WKWebView doesn't work (DigiD, iDEAL payments, OAuth),
LinkLook detects these flows automatically and hands off to the system silently.
This is not a user-facing option — it is invisible smart handoff.

### Safari look and feel

LinkLook must look and feel like Safari on iPhone. Same navigation patterns, same
toolbar placement, same gestures (swipe-to-go-back, pull-to-refresh, long-press
context menus). Users switching from Safari should feel at home immediately. The
safety check is the differentiator, not the browser chrome.

### Why this works

- **The interaction is consistent.** Users learn one simple rule: enter link →
  get verdict → choose what to do next. That's especially good for older or
  less technical users.
- **The visual hierarchy is smart.** A large "Go Back" button on riskier outcomes
  clearly nudges the safer choice without fully removing user control.
- **It separates two very different actions cleanly.** Link visiting and web
  searching go through different paths. That makes the omnibox understandable.
- **Users stay protected.** Because there is no exit to another browser, every
  link the user encounters is checked. This is especially important for threats
  that come via search results and ads (e.g. fake service company websites).

---

## Step 1: Input

The user provides input via the omnibox. LinkLook determines what it is:

### If it's a URL
- Has a scheme (`http://`, `https://`), or
- Looks like a domain (contains dots, TLD pattern)

→ Run the safety check engine (see "Check Pipeline" below) → Show the checking
  spinner → Proceed to Step 2.

### If it's a search query
- No scheme, no dots, or clearly a search phrase (multiple words, question format)

→ Open search results in LinkLook's built-in browser. No verdict screen.

### Entry points (priority order)
All entry points feed into Step 1 identically. The flow after entry is always
the same.

| Priority | Entry Point           | Description                                                  | Release    |
|----------|-----------------------|--------------------------------------------------------------|------------|
| 1        | Omnibox (type/paste)  | Core flow. Type or paste a URL into the omnibox.             | v1.0       |
| 2        | QR code scanner       | Built-in scanner. Decoded URL enters the same flow.          | v1.x       |
| 3        | Share Sheet           | iOS Share Sheet: "Check with LinkLook" from any app.         | v1.x       |
| 4        | Screenshot analysis   | Import image → AI extracts links and text → enters flow.     | v2.0 (Pro) |
| 5        | Voice input           | Microphone on omnibox → iOS speech recognition → enters flow.| v2.0       |

---

## Step 2: Verdict + Action

After the safety check completes, the user sees a verdict screen. They **must**
tap a button. There is no auto-proceed, no timeout-to-navigation, no bypass.

### No-Preload Principle

LinkLook NEVER loads, fetches, or renders the destination page before the user
explicitly taps "Open". The verdict screen
is a neutral pre-open checkpoint showing only information derived from the URL
itself and safe external lookups (GSB hash-prefix queries):

- **Domain name** — extracted from the URL, displayed prominently in large readable text
- **Full URL** — secondary, small monospace, truncated if long
- **Verdict + reason** — from the decision engine
- **Signal badges** — from structural analysis

Page titles, favicons, Open Graph metadata, and other server-provided data are
NOT fetched. Any HTTP request to the destination would leak the user's intent
and IP address to a potentially malicious server. The domain alone is the right
level of context for the user's decision. This principle is non-negotiable.

### Button labels (canonical)

| Action               | Label                          | Code action     |
|----------------------|--------------------------------|-----------------|
| Go back to safety    | **Go Back**                    | `.goBack`       |
| Open the page        | **Open**                       | `.openInApp`    |

These exact labels are used everywhere. "Open" is simple and unambiguous —
LinkLook is the browser, so "Open" naturally means "open here." There is no
`.openInBrowser` action. The old "Continue in LinkLook" / "Open in Browser"
split has been removed.

---

### Verdict: OK (Safe)

The link passed all checks.

```
┌──────────────────────────────────────┐
│                                      │
│         ✅ Link Looks Safe           │
│                                      │
│    "This is a trusted website."      │
│                                      │
│  ┌──────────────────────────────────┐│
│  │      apple.com                   ││
│  │  https://www.apple.com/store     ││
│  └──────────────────────────────────┘│
│   (domain large, full URL small)     │
│                                      │
│  ┌──────────────────────────────────┐│
│  │             OPEN                 ││
│  └──────────────────────────────────┘│
│           (large button)             │
│                                      │
└──────────────────────────────────────┘
```

- One **large** "Open" button. Opens the page in LinkLook.
- **No "Go Back" button.** The Home tab serves as the back path if the user
  changes their mind.
- **No "Open in other browser" button.** LinkLook is the browser.
- Visual treatment: green accent, positive tone.
- **Important:** The OK screen must still feel like a checkpoint — fast, clean,
  reassuring — not friction for friction's sake.

### Verdict: INFORM (Heads Up)

Minor concerns — the user should be aware but can proceed.

```
┌──────────────────────────────────────┐
│                                      │
│         ℹ️ Heads Up                  │
│                                      │
│    "LinkLook could not determine     │
│     where this shortened link        │
│     leads."                          │
│                                      │
│  ┌──────────────────────────────────┐│
│  │      short.link                  ││
│  │  https://short.link/abc123       ││
│  └──────────────────────────────────┘│
│   (domain large, full URL small)     │
│                                      │
│    [Shortened link] [New domain]     │
│                                      │
│  ┌──────────────────────────────────┐│
│  │           GO BACK                ││
│  └──────────────────────────────────┘│
│        (large, dominant)             │
│                                      │
│  ┌──────────────────────────────────┐│
│  │    Check Message Context 🔍      ││
│  └──────────────────────────────────┘│
│       (medium, outlined, Pro)        │
│                                      │
│  ┌──────────────────────────────────┐│
│  │             Open                 ││
│  └──────────────────────────────────┘│
│            (small)                   │
│                                      │
│   Share verdict with…                │
│                                      │
└──────────────────────────────────────┘
```

- **Large "Go Back"** button — visually dominant, the recommended path.
- **Medium "Check Message Context"** — outlined style, Pro feature badge. Opens
  the context analysis flow (see below).
- One **small** "Open" button — opens in LinkLook.
- Visual treatment: **blue/informational** accent. Calm, not alarming.
- Signal badges visible (up to 3).

### Verdict: WARN (Be Careful)

Significant concerns — the user is strongly nudged away.

```
┌──────────────────────────────────────┐
│                                      │
│         ⚠️ Be Careful               │
│                                      │
│    "This link uses a raw IP          │
│     address instead of a website     │
│     name."                           │
│                                      │
│  ┌──────────────────────────────────┐│
│  │      192.168.1.1                 ││
│  │  http://192.168.1.1/login.php    ││
│  └──────────────────────────────────┘│
│   (domain large, full URL small)     │
│                                      │
│    [Raw IP] [Credential keywords]    │
│                                      │
│  ┌──────────────────────────────────┐│
│  │           GO BACK                ││
│  └──────────────────────────────────┘│
│        (large, dominant)             │
│                                      │
│  ┌──────────────────────────────────┐│
│  │    Check Message Context 🔍      ││
│  └──────────────────────────────────┘│
│       (medium, outlined, Pro)        │
│                                      │
│            Open                      │
│        (very small)                  │
│                                      │
│   Share verdict with…                │
│   This doesn't look right            │
│                                      │
└──────────────────────────────────────┘
```

- Same button structure as INFORM but with **orange/caution** visual treatment.
- Colors, icons, and wording must **clearly distinguish WARN from INFORM**.
  The button layout is similar; the surrounding context does the differentiating.
- WARN "Open" button is **very small** (0.6x font, light color) to make
  "Go Back" clearly dominant. INFORM "Open" button is **small** (0.7x).
- On WARN, tapping "Open" requires an **extra confirmation tap** ("Are you sure?
  This link has warning signs.") to add friction before proceeding.
- Signal badges visible (up to 3).

### Verdict: BLOCK (Link Blocked)

Clearly malicious — no forward path.

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
- Always show the **reason** for blocking in a **prominent red card** — medium-weight
  text on a red-tinted background. Never block without explanation.
- "This doesn't look right?" false-positive report link remains.
- Visual treatment: **red/danger** accent.

### Verdict: UNKNOWN (Could Not Check)

Engine could not complete the check.

- Same as BLOCK layout: large "Go Back" button only, plus "Retry" option.
- Visual treatment: **gray/neutral**.

---

## Context Analysis Flow (Pro Feature)

Available on INFORM and WARN verdict screens via the "Check Message Context" button.

### Purpose
URL analysis alone has limits. A fresh phishing domain with valid SSL can look
clean. But the MESSAGE around the link almost always contains social engineering
tells. Context analysis catches what URL checks miss.

### Flow

1. User taps "Check Message Context" on an INFORM or WARN verdict screen.
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
6. User returns to the verdict screen with the context result visible as
   additional information. The original buttons remain unchanged.

### Data handling
- Text is processed and scored, then **immediately discarded**. Never stored.
- Must be disclosed in App Store privacy nutrition labels.
- On-device analysis preferred where possible; cloud AI as opt-in fallback.

---

## Additional Features

### "Ask Someone You Trust" (INFORM / WARN screens)
On INFORM and WARN verdict screens, a secondary "Share verdict with…" option
lets the user send the URL, verdict, and analysis summary to a trusted contact
via iMessage or WhatsApp. Not a primary button — appears as a link or icon.

### QR Code Scanner
Built-in camera-based QR code scanner. Decoded URL enters the exact same
two-step flow. Accessed from a camera icon on the Home screen or omnibox.


### Share Sheet Integration
iOS Share Sheet extension: "Check with LinkLook". Available when the user
long-presses a link in any app. Feeds the URL into Step 1. Useful when the
user is already suspicious and deliberately avoids tapping the link.

### Screenshot / Image Analysis (Pro)
User imports a screenshot of a suspicious message. AI extracts text and visible
links from the image, then runs both URL analysis and context analysis.
Combines visual and textual signals. Strong Pro tier differentiator.

### Voice Input
Microphone button on the omnibox. Uses iOS native speech recognition to capture
a URL or search query. Accessibility feature for elderly users and users with
motor difficulties.

---

## Free / Pro Tiers

LinkLook launches in three phases. The tier strategy ensures early users get the
full experience while setting honest expectations about future pricing.

### Phase 1: Early Access (v1.0 launch)

All features are unlocked for every user. No paywall, no StoreKit, no feature
gates. Pro features display a small **"Free during Early Access"** badge.

```
┌──────────────────────────────────────┐
│                                      │
│         ⚠️ Be Careful               │
│                                      │
│    suspicious-link.xyz               │
│                                      │
│  ┌──────────────────────────────────┐│
│  │           GO BACK                ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │    Check Message Context 🔍      ││
│  │         Free during Early Access ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │             Open                 ││
│  └──────────────────────────────────┘│
│            (small)                   │
│                                      │
└──────────────────────────────────────┘
```

**Why:** Maximizes testing coverage, feedback volume, and word-of-mouth.
Users experience the full product and can give informed feedback on
whether context analysis is valuable. Sets expectations for future pricing.

### Phase 2: Pro launches (post-launch update)

Early Access ends. Pro features become gated behind a subscription.
Free users see Pro features as **visible but locked**.

```
INFORM / WARN verdict screen (Free user, Phase 2):

┌──────────────────────────────────────┐
│                                      │
│         ⚠️ Be Careful               │
│                                      │
│    suspicious-link.xyz               │
│                                      │
│  ┌──────────────────────────────────┐│
│  │           GO BACK                ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │  🔒 Check Message Context   Pro  ││
│  │      (dimmed, tappable)          ││
│  └──────────────────────────────────┘│
│         ↓ tapping shows:             │
│  ┌──────────────────────────────────┐│
│  │ Context analysis is a Pro        ││
│  │ feature. It checks the message   ││
│  │ around a link for scam patterns  ││
│  │ like urgency, pressure, and      ││
│  │ impersonation.                   ││
│  │                                  ││
│  │        [ Upgrade ]               ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │             Open                 ││
│  └──────────────────────────────────┘│
│            (small)                   │
│                                      │
└──────────────────────────────────────┘
```

**Why locked-but-visible?** The upgrade prompt appears at the moment of
highest user intent — when they're staring at a WARN verdict and feeling
uncertain. That's the best conversion moment. Hiding the feature entirely
means users never discover it.

### What is always Free

The entire core safety product and browser:

- Two-step safety flow (enter link → verdict → open in LinkLook)
- Structural analysis + Google Safe Browsing checks
- All verdict screens (OK, INFORM, WARN, BLOCK)
- All entry points (omnibox, QR, clipboard, Share Sheet)
- Full browser features: tabs, favorites, reader mode, downloads
- History and Recent tab
- Voice input
- Password AutoFill (via iOS system keychain — no code needed)

**The safety check and the browser are what makes people install and stay.
Both are never paywalled.**

### What is Pro

AI-powered features that add analysis on top of the free safety flow:

- **On-device AI URL analysis** (Core ML model adds a third signal to the check pipeline)
- **Context analysis** ("Check Message Context" button)
- **Screenshot/image analysis** (import suspicious message screenshots)
- **"AI-enhanced protection"** label during checking spinner (vs "Basic protection")

### Per-screen differences (Phase 2)

| Verdict  | Free user                                          | Pro user                              |
|----------|----------------------------------------------------|-----------------------------------------|
| OK       | Structural + GSB only                              | Structural + GSB + AI URL analysis      |
| INFORM   | Structural + GSB; "Check Message Context" locked 🔒| All three signals; context analysis     |
| WARN     | Structural + GSB; "Check Message Context" locked 🔒| All three signals; context analysis     |
| BLOCK    | Identical                                          | Identical                               |
| UNKNOWN  | Identical                                          | Identical                               |
| Checking | "Basic protection"                                 | "AI-enhanced protection"                |

---

## Check Pipeline (Step 1 Internals)

When a URL enters Step 1, the decision engine runs checks **in parallel**:

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
│ • TLD check  │ │ • Threat types:│ │   model      │    │
│ • IP detect  │ │   MALWARE,     │ │ • Trained on │    │
│ • Brand      │ │   SOCIAL_ENG,  │ │   phishing   │    │
│   lookalike  │ │   UNWANTED_SW, │ │   datasets   │    │
│ • Redirect   │ │   HARMFUL_APP  │ │ • Scores URL │    │
│   chains     │ │ • 3s timeout   │ │   features   │    │
│ • Keyword    │ │                │ │ • On-device  │    │
│   analysis   │ │                │ │   only       │    │
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
          Step 2: Verdict
```

### LinkLook structural analysis
The existing on-device engine: domain patterns, TLD reputation, raw IP detection,
brand lookalike matching, redirect chain analysis, keyword scoring, character
substitution detection. This is what LinkLook already does today.

### Google Safe Browsing (GSB)
Checks the URL against Google's threat database. GSB catches **known** threats
(reported phishing sites, confirmed malware hosts) that structural analysis can't
detect — a perfectly-formed URL on a known-bad domain will look clean to structural
checks but GSB will flag it.

**How it works:**
- Uses the GSB **Lookup API v4** (hash-prefix based).
- Sends SHA-256 hash prefixes of the URL to Google, **not** the full URL.
- Checks threat types: `MALWARE`, `SOCIAL_ENGINEERING`, `UNWANTED_SOFTWARE`,
  `POTENTIALLY_HARMFUL_APPLICATION`.
- Timeout: **3 seconds**. If GSB doesn't respond in time, proceed with structural
  analysis only.

**How GSB results affect the verdict:**
- **GSB match** → Escalate to BLOCK. Reason: "This site has been flagged as
  dangerous by Google Safe Browsing." Reason code: `gsbThreatMatch`.
- **GSB clean** → No effect. Structural analysis verdict stands. A clean GSB
  result never downgrades a WARN or INFORM.
- **GSB timeout** → No effect on verdict. Add `externalLookupFailed` signal
  (INFORM level). Never block because GSB timed out. Never mark safe because
  GSB returned clean.

**Why both checks matter:**
- Structural analysis catches **new/unknown** threats: fresh phishing domains,
  suspicious patterns, brand impersonation attempts that haven't been reported yet.
- GSB catches **known** threats: confirmed phishing sites, malware hosts, sites
  reported by users and security researchers.
- Together they cover both novel and established threats. Neither alone is sufficient.

**Privacy:**
- GSB Lookup API sends hash prefixes, not full URLs. Google cannot reconstruct
  the exact URL from the prefix.
- Must be disclosed in App Store privacy nutrition labels and the privacy policy.

**API key handling:**
- The GSB API key must NOT be stored in source code.
- Use Xcode build configuration or a server-side proxy.
- Cache GSB results per URL for 5 minutes to avoid redundant calls on retry.

### On-device AI URL analysis (Pro only)

A Core ML classification model that runs entirely on-device. It scores URLs on
features that rule-based structural analysis may miss — subtle combinations of
signals that individually look innocent but together form patterns the model has
learned from thousands of real phishing URLs.

**What it analyzes:**
- Subdomain length and depth
- TLD category and reputation
- Path structure and depth
- Character entropy and distribution
- Keyword patterns and combinations
- Overall URL "shape" compared to known phishing patterns

**How it works:**
- Runs via **Core ML** on the device. No data leaves the phone. Full privacy.
- Trained on public phishing datasets (PhishTank, OpenPhish) + legitimate URL corpus.
- Returns a risk score. If the score exceeds the threshold, it contributes signal
  `aiUrlRiskHigh` which can escalate the verdict.
- Must complete within the same 8-second checking window. If slow, structural +
  GSB results are used without it.

**How AI URL results affect the verdict:**
- **High risk score** → Can escalate (OK→INFORM, INFORM→WARN). Reason:
  "LinkLook's AI model detected suspicious patterns in this web address."
- **Low risk score** → No effect. Never downgrades a verdict from structural
  analysis or GSB.
- **Timeout/failure** → No effect. Graceful degradation to structural + GSB.

**Why this matters:**
- Structural analysis catches patterns someone explicitly coded as rules.
- GSB catches URLs that have been reported and confirmed as malicious.
- AI URL analysis catches **the gap between the two**: URLs that don't match
  any explicit rule and haven't been reported yet, but whose *combination*
  of features looks like phishing. This is especially valuable for fresh
  phishing domains that are hours or days old.

**Pro only:** Free users get structural analysis + GSB. Pro users get all three
signals merged. This is the "AI-enhanced protection" that the checking spinner
labels refer to.

**Model updates:** Ship with app updates. No over-the-air model replacement
in v1.x. Model must be validated against LinkLook's existing test fixtures
before each release.

### No safe-URL caching (Design Decision)

LinkLook runs a **fresh check on every URL, every time**. It never skips or
shortens a check based on a previous safe result.

**Why:** A URL that was safe yesterday might not be safe today. Domains get
compromised, clean redirects get swapped for malicious ones, legitimate sites
get injected with malware. The entire product promise is "we check before you
go." Caching safe results undermines that promise.

**The only caching allowed:** GSB result caching (5 minutes) to avoid redundant
API calls on retry/re-check. This is an **API optimization**, not a verdict
cache — the full check pipeline (structural analysis + signal merge) still runs
every time. The GSB cache just reuses a recent API response instead of calling
Google again for the same URL within 5 minutes.

**History is not a cache.** The Recent tab stores what the user checked and
what the verdict was. Tapping a history entry re-runs the check. It gives quick
access to previously visited URLs while still running a fresh check every time.

---

## Design Principles

1. **The forced pause is the product.** Never shortcut Step 2.
2. **Button hierarchy communicates risk without words.** A screen with one giant
   button vs. one small one is universally understood.
3. **Colors and context differentiate verdicts, not button layout.** INFORM and
   WARN have similar buttons; the visual treatment tells the story.
4. **Every entry point converges on the same flow.** QR, clipboard, share sheet,
   voice — they all feed into Step 1. The experience after entry is identical.
5. **Respect the user's ability to decide.** Even on WARN, offer a path forward.
   Only BLOCK removes the choice entirely.
6. **Design for the elderly audience.** Large tap targets, clear labels, no
   jargon, no ambiguity. When in doubt, make it bigger and simpler.
7. **OK must feel like a checkpoint, not a speed bump.** Fast, clean, reassuring.
8. **Look like Safari.** Users should feel at home. The safety check is the
   differentiator, not the browser chrome.
9. **Keep users in.** No exit to other browsers. Protection only works if
   LinkLook handles every link.

---

## Summary

> **Two-step safety flow:** Every entered or tapped URL in LinkLook is handled
> in two steps: (1) check the destination, (2) let the user explicitly choose
> what happens next. Users never land on a URL automatically. LinkLook always
> shows a verdict first. Only after that does the user decide to go back or
> open the page — always inside LinkLook. There is no option to open in another
> browser. This keeps users protected at all times.

---

## What LinkLook Is NOT

To keep the product focused, LinkLook does **not** include:

- VPN functionality (different product category, dilutes the message)
- Its own password manager (iOS system keychain + third-party managers work
  automatically in WKWebView — no code needed)
- Full ad/tracker blocking (shifts positioning to "content blocker")
- Automatic background scanning of all browsing (invasive, battery-draining;
  the two-step model is better because it's intentional and user-initiated)
- An "Open in other browser" option (users must stay in LinkLook for protection
  to work; edge cases like DigiD/iDEAL are handled via automatic system handoff)

---

## Safari Extension: Second Defense Layer

LinkLook's primary protection requires being the default browser. But not every
user will switch immediately. For users still browsing in Safari, LinkLook
provides a **Safari Web Extension** and a **Content Blocker** as a second line
of defense.

### What the Safari extension does

The Web Extension runs inside Safari and provides:

- **Search result warnings** — marks suspicious links on Google/Bing result pages
- **Landing page banners** — warning banner on pages with suspicious domains
- **"Open in LinkLook" button** — sends the URL to LinkLook for a full check
- **Domain reputation badges** — small safety indicator on every page

The Content Blocker provides hard-block rules for confirmed malicious domains.
It runs independently (Safari applies the rules — the blocker never sees which
pages the user visits).

### How it connects to the two-step flow

The extension is NOT a replacement for LinkLook-as-browser. It is a bridge:

1. User browses in Safari (not yet switched to LinkLook)
2. Extension warns about suspicious search results or pages
3. User taps "Open in LinkLook" → enters the full two-step safety flow
4. Over time, users learn to trust LinkLook and switch their default browser

The extension creates awareness and builds trust. The browser is the product.

### Activation challenge

Users must manually enable the extension in iOS Settings → Apps → Safari →
Extensions. This is a significant friction point, especially for elderly users.
LinkLook's onboarding must include a clear, step-by-step activation guide.

Full spec: `CLAUDE.md` rules 62–74.
