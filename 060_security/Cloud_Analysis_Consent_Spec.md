# Cloud Analysis and Safe Preview Consent Spec

> **Status:** Approved — implementation in progress.
> **Date:** 2026-03-16
> **Scope:** Defines the user-facing, technical, and policy design for cloud URL
> analysis, server-side safe preview, and server-side page AI analysis.

## Executive Decision

Combine at the feature-family level, separate at the permission and service level.

- In the app, present these capabilities as part of one understandable protection family.
- Under the hood, keep them as separate scopes, separate backend endpoints, and separate policy decisions.

## Core Principle

LinkLook always prefers the least invasive path first:

**On-device check → cloud URL check → server-side page fetch/preview only when needed and allowed.**

## Three Distinct Privacy Actions

| # | Action | What it sends | Scope key |
|---|--------|---------------|-----------|
| 1 | Cloud URL analysis | URL + minimal metadata | `url_analysis` |
| 2 | Server-side safe preview | Server fetches + renders the page | `page_fetch` |
| 3 | Server-side page AI analysis | Server inspects page content | `page_ai_analysis` |

These are **not** the same privacy action and must not be bundled behind one vague switch.

## User-Facing Structure

### Settings → Enhanced Cloud Protection

**A. Cloud URL Check**
> Send the web address to LinkLook's server for a stronger scam and risk check.

**B. Safe Preview and Deep Analysis**
> Allow LinkLook to fetch the page on the server to create a safe preview or
> inspect page content without opening the site directly on your device.

### Settings Screen Layout

```
Settings > Privacy and Protection

  Protection mode

    On-device checks
    Always on — Checks links locally on your device.

    Cloud URL Check               [toggle]
    Send web addresses to LinkLook's server for stronger risk analysis.

    Safe Preview and Deep Analysis [toggle]
    Allow LinkLook to fetch pages on the server to create a safe preview
    or inspect page content without opening the site directly on your device.

    Ask before server-side page fetch [toggle, default ON]
    Show a confirmation before LinkLook fetches a page on the server.

    Auto-use cloud help for unclear results [toggle]
    Use cloud URL analysis automatically for unclear or suspicious results.

    Clear cloud-analysis history   [action]
    Delete locally visible history and trigger server-side deletion where applicable.
```

### Default Values

| Setting | First launch | After URL check enabled | After deeper enabled |
|---------|-------------|------------------------|---------------------|
| On-device checks | ON | ON | ON |
| Cloud URL Check | OFF | ON | ON |
| Safe Preview + Deep Analysis | OFF | OFF | ON |
| Ask before page fetch | ON | ON | ON |
| Auto-use cloud for unclear | OFF | OFF → ON for INFORM/WARN | ON |

## First-Run Consent Sheet

**Title:** Choose your protection level

**Options:**

1. **On-device only** — Your links are checked on your device. No URL is sent to LinkLook for cloud analysis.
2. **Cloud URL Check** — LinkLook may send the web address to our server for a stronger risk check.
3. **Cloud URL Check + Safe Preview** — LinkLook may also fetch the page on our server to create a safe preview or inspect page content without opening the site directly on your device.

**Buttons:** Keep on-device only / Allow URL check only / Allow both

## Just-in-Time Prompt (First Preview Tap)

**Title:** Allow safe preview?

**Body:** To create a safe preview, LinkLook needs to fetch the page on our
server instead of opening it directly on your device.

**Buttons:** Not now / Allow once / Always allow

## Runtime Decision Ladder

1. **On-device analysis** — Always runs first.
2. **Cloud URL analysis** — Only if `url_analysis` scope is granted and local result is inconclusive or policy says cloud is useful.
3. **Page fetch / preview** — Only if `page_fetch` scope is granted AND LinkLook needs it for preview or deeper analysis.

### Behavior by Verdict

| Verdict | Cloud URL check | Page fetch/preview |
|---------|----------------|-------------------|
| OK | Not needed | Not needed |
| INFORM | Auto if enabled | Only if user requests preview |
| WARN | Auto if enabled | Only if user requests preview |
| BLOCK | May run | **No automatic fetch. No preview.** |

## Backend Endpoints

| Endpoint | Scope required | Purpose |
|----------|---------------|---------|
| `/analyze-url` | `url_analysis` | URL-only risk scoring |
| `/fetch-preview` | `page_fetch` | Server-side screenshot |
| `/analyze-page` | `page_ai_analysis` | Page content AI analysis |

Server-side enforcement: reject operations outside granted scope.

## Consent Rules

1. Cloud features are opt-in.
2. Page fetch must not be silently implied by URL analysis consent.
3. User can revoke either layer at any time.
4. No hidden escalation from URL-only to page fetch.
5. Keep explanations plain — no jargon.

## Logging and Retention

- **URL analysis:** Store only what's needed for real-time verdicting, abuse prevention, and reliability.
- **Page fetch / preview:** Store only what's needed for immediate rendering and short-lived safety analysis.
- Separate retention classes: short for transient fetch artifacts, longer only for security logs if needed.
- Avoid retaining full page content by default.

## Privacy Policy Structure

1. **On-device checks** — Core link checks on device. No data leaves.
2. **Cloud URL analysis** — If enabled, URL + limited metadata sent to server.
3. **Safe preview and deeper analysis** — If enabled, server fetches page for preview/inspection.
4. **User control** — Users can toggle on/off in Settings. Revocation is immediate.

## v1 Simplification

For v1, expose two user choices (internally three scopes):

- **Choice A:** Cloud URL Check (`url_analysis`)
- **Choice B:** Safe Preview and Deep Analysis (`page_fetch` + `page_ai_analysis`)
