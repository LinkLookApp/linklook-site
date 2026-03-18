# Cloud Analysis and Safe Preview Consent Spec

> **Status:** Approved — v2 (three-toggle model).
> **Date:** 2026-03-16
> **Supersedes:** v1 two-toggle model ("Cloud URL Check" + "Safe Preview and Deep Analysis").
> **Scope:** Defines the user-facing, technical, and policy design for cloud URL
> analysis, server-side safe preview, and server-side page AI analysis.

## Executive Decision

Three independent toggles, one protection family.

- In the app, present these capabilities as part of one understandable section ("Enhanced Cloud Protection").
- Each toggle controls exactly one privacy action. No hidden bundling.
- Under the hood, each toggle maps to one scope and one backend endpoint.
- Toggles are independent: the user can enable any combination.

### Why three toggles (not two)

The v1 spec bundled Safe Preview and Deep Page Analysis into one toggle
("Safe Preview and Deep Analysis"). This was a pragmatic simplification but
caused two problems:

1. **Blurred consent.** A user who just wants to see the page visually is also
   consenting to AI content inspection — without realizing it.
2. **Backend scope leak.** The `/analyze-url` endpoint was calling
   `fetch_page_content()` + `analyze_page()` behind the scenes, violating the
   "no hidden escalation" consent rule.

The three-toggle model fixes both: each toggle = one privacy action = one
endpoint = one scope. Clear for the user, clear for Apple, clear in the code.

## Core Principle

LinkLook always prefers the least invasive path first:

**On-device check → cloud URL check → server-side page fetch → page AI analysis — only when needed and allowed.**

## Three Distinct Privacy Actions

> **Naming convention:** The user-facing label in the app is "Cloud link check" (simple).
> The technical/spec name is "Cloud URL Check" (maps to `url_analysis` scope).
> Both refer to the same feature. Use the user-facing name in all UI text.

| # | User-facing label | Technical name | What happens | What leaves the device | Scope key | Backend endpoint |
|---|------------------|---------------|-------------|----------------------|-----------|-----------------|
| 1 | Cloud link check | Cloud URL Check | AI analyzes the web address only | URL + minimal metadata | `url_analysis` | `/analyze-url` |
| 2 | Safe Preview | Server fetches the page and returns a screenshot | URL (server fetches page) | `page_fetch` | `/fetch-preview` |
| 3 | Deep Page Analysis | AI reads the page content for scam signs | URL (server fetches + inspects page) | `page_ai_analysis` | `/analyze-page` |

These are **three separate privacy actions**. Each has its own consent toggle,
its own scope, and its own backend endpoint. No bundling.

### What the user gets from each

| Toggle | User benefit | Privacy cost |
|--------|-------------|-------------|
| Cloud URL Check | Stronger scam detection from the URL alone. Catches things the on-device engine misses. | URL sent to LinkLook's server. Page is NOT fetched. |
| Safe Preview | See the page visually without loading it on your device. | LinkLook's server visits the page and takes a screenshot. |
| Deep Page Analysis | **Strongest protection.** AI reads the page looking for fake login forms, brand impersonation, urgency language, scam patterns. | LinkLook's server visits the page AND reads its content. |

The progression is: URL only → see the page → read the page. Each step is more
invasive and more powerful. The user decides how far to go.

## User-Facing Structure

### Settings → Enhanced Cloud Protection

```
Settings > Privacy and Protection

  Protection mode

    On-device checks
    Always on — Checks links locally on your device.

    Cloud URL Check                    [toggle]
    AI analyzes the web address only.
    Catches risks the on-device check may miss.

    Safe Preview                       [toggle]
    See the page without opening it on your device.
    LinkLook's server takes a screenshot for you.

    Deep Page Analysis                 [toggle]
    AI reads the page for scam signs.
    Strongest protection — detects fake login forms,
    brand impersonation, and social engineering.

    ─────────────────────────────────────────────

    Ask before fetching a page         [toggle, default ON]
    Show a confirmation before LinkLook fetches a page
    on the server. Applies to Safe Preview and
    Deep Page Analysis.

    Auto-use cloud URL check           [toggle]
    Automatically send unclear or suspicious URLs
    for cloud analysis.

    Clear cloud-analysis history       [action]
    Delete locally visible history and trigger
    server-side deletion where applicable.
```

### Toggle independence

- Each toggle can be enabled or disabled independently.
- Enabling Deep Page Analysis does NOT auto-enable Safe Preview (they're
  independent — a user may want AI analysis without seeing the page).
- Enabling Safe Preview does NOT auto-enable Deep Page Analysis.
- Enabling either Safe Preview or Deep Page Analysis auto-enables Cloud URL
  Check (the lightest scope) since the heavier scopes imply willingness.
- "Ask before fetching a page" applies to both Safe Preview and Deep Page
  Analysis (anything involving `page_fetch`).

### Default Values

| Setting | First launch | After URL check enabled | After preview or deep enabled |
|---------|-------------|------------------------|------------------------------|
| On-device checks | ON | ON | ON |
| Cloud URL Check | OFF | ON | ON (auto-enabled) |
| Safe Preview | OFF | OFF | (whichever was toggled) |
| Deep Page Analysis | OFF | OFF | (whichever was toggled) |
| Ask before page fetch | ON | ON | ON |
| Auto-use cloud URL check | OFF | ON for INFORM/WARN | ON |

## First-Run Consent Sheet

**Title:** Choose your protection level

**Options:**

1. **On-device only** — Your links are checked on your device. Nothing is sent to LinkLook's server.
2. **Cloud URL Check** — LinkLook sends the web address to our server for a stronger risk check. The page itself is not fetched.
3. **Full cloud protection** — LinkLook can also fetch the page on our server to create a safe preview and detect scam content. This gives the strongest protection.

**Buttons:** Keep on-device only / URL check only / Full protection

**Note:** The first-run sheet sets initial toggle state. The user can change any
toggle in Settings at any time. "Full protection" enables all three toggles.

## Cloud Upgrade Privacy Confirmation (CLAUDE.md rule 68)

**Principle: every time more data goes to the cloud, confirm first.**

When the user selects a cloud option (in onboarding OR in Settings), a
`CloudPrivacyConfirmationSheet` appears explaining the privacy impact before
the setting takes effect. The sheet has "Turn on" / "Cancel" buttons.

### Two variants

| Level | What the sheet says | Icon |
|-------|-------------------|------|
| Cloud link check | "The link you check is sent to the LinkLook cloud for AI analysis." + "The website behind the link is not fetched or opened." | `cloud` |
| Cloud link & page check | "The link is sent to the LinkLook cloud, and the website behind it is fetched and analyzed there." + "The page is only viewed by LinkLook's server — it is not stored permanently." | `cloud.fill` |

### When confirmation IS required

- On-device only → Cloud link check
- On-device only → Cloud link & page check
- Cloud link check → Cloud link & page check (any toggle that adds page-level scope)

### When confirmation is NOT required

- Downgrading: Cloud → On-device only
- Downgrading: Cloud link & page check → Cloud link check
- Re-selecting the same level that is already active

### Where it applies

1. **Onboarding** (`CloudProtectionConsentSheet`) — tapping a cloud option
   triggers the privacy confirmation as a sub-sheet before the choice is
   applied.
2. **Settings** (`SettingsView`) — toggle bindings intercept the ON action
   and present the confirmation sheet. If the user cancels, the toggle stays
   OFF. Turning toggles OFF never shows a confirmation.

### Accessibility identifiers

- `privacyConfirmButton` — "Turn on"
- `privacyCancelButton` — "Cancel"

## Just-in-Time Prompts

### First Safe Preview tap

**Title:** Allow safe preview?

**Body:** To show a preview, LinkLook needs to fetch the page on our server
instead of opening it on your device. You'll see a screenshot — nothing runs
on your iPhone.

**Buttons:** Not now / Allow once / Always allow

### First Deep Page Analysis trigger

**Title:** Allow deep page analysis?

**Body:** LinkLook's AI can read the page content to detect fake login forms,
brand impersonation, and scam patterns. The page is fetched on our server —
nothing runs on your iPhone. This gives the strongest scam detection.

**Buttons:** Not now / Allow once / Always allow

## Runtime Decision Ladder

> **Two-step model:** URL checking and webpage checking are separate steps.
> The page is never fetched until the user explicitly requests it.
> Full spec: `LinkLook-Docs/050_architecture/URL_Webpage_Analysis_Model.md`.

### Step 1 — URL Phase

1. **On-device analysis** — Always runs first.
2. **Sensitive data detection** — On-device scan of URL for personal/sensitive
   data (emails, tokens, account IDs, etc.). Runs before any backend call.
   If sensitive data is found, the Privacy Signal flow activates (see step 2a).
   Full spec: `URL_Privacy_Signal_Spec.md`.
   2a. **Privacy Signal** — User chooses: privacy-protected check (masked URL),
       full check (original URL), or cancel (no backend call). The choice applies
       to all subsequent backend calls in this check session.
3. **Cloud URL analysis** — Only if `url_analysis` scope is granted. The backend
   analyzes the URL string only (masked or full per Privacy Signal choice).
   It does NOT fetch the page.

### Step 2 — Webpage Phase (explicit user action only)

4. **Safe Preview** — Only if `page_fetch` scope is granted AND the user taps
   "Preview page." Never automatic. Uses masked or full URL per Privacy Signal
   choice.
5. **Deep Page Analysis** — Only if `page_ai_analysis` scope is granted AND
   the user taps "Check webpage with AI." Never automatic. Two modes:
   - **Reduced mode (default):** AI analyzes a stripped webpage representation
     (extracted text, headings, forms, CTAs). Privacy-reduced.
   - **Full mode (explicit escalation):** AI analyzes more page content. Only
     offered when reduced analysis confidence is low, or when user explicitly
     requests deeper analysis. Audit logged.
   Uses masked or full URL per Privacy Signal choice.

### Critical rule: `/analyze-url` must be URL-only

The `/analyze-url` endpoint receives the URL string and analyzes it server-side
(domain patterns, TLD risk, AI model on URL features). It must NEVER call
`fetch_page_content()` or read the destination page. If it fetches the page,
it is doing `page_fetch` work under the `url_analysis` scope — a consent
violation.

Page fetching happens ONLY via `/fetch-preview` (screenshot) and
`/analyze-page` (AI content analysis).

### Behavior by Verdict

| Verdict | Cloud URL check | Safe Preview | Deep Page Analysis |
|---------|----------------|-------------|-------------------|
| OK | Not needed | Not needed | Not needed |
| INFORM | Auto if enabled | User taps "Preview page" | Auto if enabled, or user request |
| WARN | Auto if enabled | User taps "Preview page" | Auto if enabled, or user request |
| BLOCK | May run | **No preview** | May run for training data |

## Backend Endpoints

| Endpoint | Scope required | What it does | What it must NOT do |
|----------|---------------|-------------|-------------------|
| `/analyze-url` | `url_analysis` | Analyze URL string (structure, domain, TLD, AI model on URL features). Return risk score. | Must NOT fetch page content. Must NOT call Urlbox. |
| `/fetch-preview` | `page_fetch` | Fetch page via Urlbox, return JPEG screenshot. | Must NOT run page content analysis unless `/analyze-page` is also called. |
| `/analyze-page` | `page_ai_analysis` | Fetch page content, extract text, run AI threat analysis. Accepts `mode=reduced` (default: stripped representation) or `mode=full` (deeper analysis, audit logged). Store for SLM training. | Must NOT auto-escalate from reduced to full without user action. |

**Server-side enforcement:** Each endpoint checks the scope. Reject operations
outside granted scope. No hidden escalation between endpoints.

### Backend fix required (v2 migration)

The v1 `/analyze-url` endpoint calls `_run_analysis()` which does
`fetch_page_content()` + `analyze_page()`. This must be fixed:

1. `/analyze-url` → analyze URL string only (no page fetch).
2. Move page-fetching analysis to `/analyze-page` only.
3. If training data collection is needed for all URLs, it must go through
   `/analyze-page` with the `page_ai_analysis` scope — not hidden inside
   `/analyze-url`.

## Consent Rules

1. Cloud features are opt-in. All OFF by default.
2. Each toggle controls exactly one privacy action.
3. Safe Preview does not imply Deep Page Analysis (or vice versa).
4. Cloud URL Check does not imply page fetching.
5. No hidden escalation: an endpoint must not do work beyond its scope.
6. User can revoke any toggle at any time. Effect is immediate.
7. Keep explanations plain — no jargon.
8. **Privacy Signal gate:** When sensitive data is detected in a URL, the
   Privacy Signal dialog must be shown before any backend call — even if all
   consent toggles are enabled. Consent toggles control *whether* data goes
   to the backend; the Privacy Signal controls *how* (masked or full). Both
   layers must pass for a backend call to proceed.
9. The Privacy Signal never silently strips or sends sensitive URL data.
   The user always makes an explicit choice.

## Scope Interaction Matrix

| User has enabled | `/analyze-url` | `/fetch-preview` | `/analyze-page` |
|-----------------|----------------|------------------|-----------------|
| Nothing | Blocked | Blocked | Blocked |
| Cloud URL Check only | Allowed | Blocked | Blocked |
| Safe Preview only | Auto-enabled (URL check) | Allowed | Blocked |
| Deep Page Analysis only | Auto-enabled (URL check) | Blocked | Allowed |
| Safe Preview + Deep | Auto-enabled (URL check) | Allowed | Allowed |
| All three | Allowed | Allowed | Allowed |

## Logging and Retention

- **URL analysis:** URL string + verdict. Short retention (days). No page content.
- **Safe Preview:** URL sent to Urlbox for screenshot. Screenshot returned to user, not stored. URL discarded immediately.
- **Deep Page Analysis (reduced mode):** Stripped webpage representation sent to AI. Analysis result stored for SLM training (90-day FIFO). Page text discarded after analysis.
- **Deep Page Analysis (full mode):** Full page content sent to AI. Analysis result stored for SLM training (90-day FIFO). Page text discarded after analysis. **Audit log entry required:** pseudonymous device ID, timestamp, event type (`full_page_analysis_selected`), URL hash, analysis mode, consent version.
- Separate retention classes per scope. Never retain page content from one scope in another's storage.
- **Audit log retention:** 90 days. Log the decision, not the content. Never log raw URLs, page content, or screenshots in the audit log.

## Technical Non-Logging Controls

> Full spec: `LinkLook-Docs/050_architecture/URL_Webpage_Analysis_Model.md` §9a.
> Enforceable rules: `CLAUDE.md` rules 118–119.

Non-logging of sensitive data is enforced by architecture, not just policy:

- **Structured allowlisted logging** — only defined fields are written to logs.
- **No request-body logging** — middleware and proxies do not log request bodies.
- **Query string sanitization** — sensitive URL parameters are stripped before
  any log write.
- **Page content excluded** — page text, HTML, and screenshots are processed
  in memory and never written to application logs.
- **Crash tools configured** — Sentry/Crashlytics exclude sensitive payload
  fields from crash reports.
- **Canary tests in CI** — automated tests inject fake sensitive markers and
  verify they do not appear in any log output.

## Privacy Policy Structure

1. **On-device checks** — Core link checks on device. No data leaves.
2. **Cloud URL Check** — If enabled, URL + limited metadata sent to server. Page is not fetched.
3. **Safe Preview** — If enabled, server fetches page for screenshot only. URL discarded after render.
4. **Deep Page Analysis** — If enabled, server fetches page and AI inspects content. Analysis stored for safety improvement; page text discarded.
5. **User control** — Three independent toggles in Settings. Revocation is immediate.

## Elderly-User Language Guide

| Technical concept | User-facing language |
|------------------|---------------------|
| `url_analysis` | "AI analyzes the web address only" |
| `page_fetch` | "See the page without opening it on your device" |
| `page_ai_analysis` | "AI reads the page for scam signs" |
| Scope escalation | "Each option is independent. You choose." |
| Strongest protection | "Detects fake login forms, brand impersonation, and social engineering" |
| Reassurance | "You stay in control. You can turn this off anytime." |

## Implementation Summary

### What to build (code changes from v1)

**Settings UI:**
- Replace "Safe Preview and Deep Analysis" toggle with two separate toggles: "Safe Preview" and "Deep Page Analysis".
- Add auto-enable logic: enabling Safe Preview or Deep Page Analysis auto-enables Cloud URL Check.
- "Ask before fetching a page" applies to both Safe Preview and Deep Page Analysis.

**PreferencesStore:**
- Split `safePreviewDeepAnalysisEnabled` into `safePreviewEnabled` and `deepPageAnalysisEnabled`.
- `pageFetchPermitted` → returns `safePreviewEnabled || deepPageAnalysisEnabled`.
- `pageAIAnalysisPermitted` → returns `deepPageAnalysisEnabled` only.
- Add `safePreviewPermitted` → returns `safePreviewEnabled`.

**Backend:**
- Fix `/analyze-url`: remove `fetch_page_content()` and `analyze_page()` calls. Make it truly URL-only.
- `/fetch-preview`: unchanged (screenshot only).
- `/analyze-page`: unchanged (page fetch + Claude analysis + storage).
- Add scope validation to `verify_jwt` (future hardening).

**Runtime logic:**
- `submitForBackendAnalysis()` calls `/analyze-url` (URL-only, no page fetch).
- Safe Preview button calls `/fetch-preview` (screenshot only).
- Deep Page Analysis calls `/analyze-page` (page fetch + AI inspection).
- JIT prompts: separate prompts for first Safe Preview tap and first Deep Page Analysis trigger.

**First-run consent sheet:**
- Update to three options: On-device only / URL check only / Full protection.

### One-line conclusion

Three toggles, three scopes, three endpoints. Each does exactly what it says.
