# Safe Preview — Screenshot-Only Page Preview

**Status:** Planned (engineering handover complete)
**Date:** 2026-03-14
**Phase:** Post-launch, Pro feature
**Handover:** `LinkLook_SafePreview_Handover.docx` (March 2026)

## Problem

On the verdict screen, the user sees the domain name, full URL, verdict, and
signal badges — but not what the page actually looks like. For INFORM and WARN
verdicts, users often want to "peek" at the destination before deciding. Without
a preview, they must either trust the verdict blindly or proceed to the page.

Loading the page on the user's device to provide a preview would violate the
No-Preload Principle: it leaks the user's IP and intent, runs JavaScript, sets
cookies, and potentially triggers malicious behavior.

## Solution: Safe Preview

A backend proxy server sends the URL to a screenshot service (Urlbox), which
renders the page in a sandboxed headless browser and returns a flat JPEG image.
The user sees the page visually, but nothing runs on their device. The user's
IP is never exposed to the destination.

### Architecture

```
iOS App                  LinkLook Proxy              Urlbox
(Swift)                  (FastAPI / Fly.io)          (urlbox.io)
   │                          │                          │
   │ POST /preview            │                          │
   │ JWT auth + URL ────────► │                          │
   │                          │ HMAC-signed request ───► │
   │                          │                          │ Renders page
   │                          │                          │ in sandbox
   │                          │ ◄─── JPEG screenshot ── │
   │ ◄──── JPEG bytes ─────  │                          │
   │                          │                          │
   │ Display in verdict       │ No logging.              │
   │ screen as flat image     │ Discard immediately.     │
   └──────────────────────────┴──────────────────────────┘
```

**Why this architecture (not direct Urlbox from iOS):**

- The Urlbox API key cannot be embedded in the iOS binary — it would be
  extractable by any security researcher in minutes.
- The proxy keeps credentials server-side and adds URL validation, rate
  limiting, and JWT authentication between the app and the service.

**Why Urlbox (not self-hosted Playwright) for v1:**

- Zero ops burden at launch — no headless browser fleet to maintain.
- One API key, one DPA, live in days not weeks.
- Urlbox Starter (~€18/month) covers 1,000 renders/month — sufficient for
  early user base.
- Self-hosted Playwright is the v2 upgrade path if Urlbox becomes a cost or
  control constraint.

### How it works

1. User taps "Preview page" on the verdict screen (INFORM or WARN only).
2. iOS app signs a JWT using the shared secret and POSTs the URL to `/preview`
   on the proxy.
3. Proxy validates the JWT, validates the URL (rejects private IPs, non-HTTP/S
   schemes).
4. Proxy calls Urlbox API with HMAC-signed request. JS disabled, ads blocked,
   390×844px viewport (iPhone dimensions), JPEG output at 80% quality.
5. Urlbox returns a JPEG screenshot. Proxy forwards bytes to the iOS app with
   no logging.
6. iOS app renders the screenshot in the verdict screen preview pane.
7. User decides: Go Back or Open.

### What the user sees

```
┌──────────────────────────────────────┐
│         ℹ️ Heads Up                  │
│                                      │
│    "This domain is unfamiliar."      │
│                                      │
│  ┌──────────────────────────────────┐│
│  │      example-shop.xyz            ││
│  │  https://example-shop.xyz/offer  ││
│  └──────────────────────────────────┘│
│                                      │
│  ┌──────────────────────────────────┐│
│  │                                  ││
│  │   [Screenshot of the page]       ││
│  │   (non-interactive image)        ││
│  │                                  ││
│  └──────────────────────────────────┘│
│  "Preview only — page not loaded     │
│   on your device"                    │
│                                      │
│  [           GO BACK                ]│
│  [    Check Message Context 🔍      ]│
│  [            Open                  ]│
└──────────────────────────────────────┘
```

### Security properties

| Property | Guarantee |
|----------|-----------|
| No JavaScript on user device | The page runs only in the Urlbox sandbox |
| No cookies set on user device | The user's browser state is untouched |
| No IP leak to destination | The destination sees LinkLook's proxy/Urlbox IP, not the user's |
| No tracking pixels fire on user device | All tracking executes in the sandbox only |
| No downloads on user device | The proxy does not forward any downloads |
| Image is flat | JPEG — no embedded scripts, no active content |
| Sandbox is disposable | Each render uses a fresh environment |
| No URL logging | Proxy discards URL immediately after returning screenshot |

### Infrastructure

| Component | Choice |
|-----------|--------|
| Runtime | Python 3.12 + FastAPI + Uvicorn |
| Hosting | Fly.io — auto-scale to zero, EU region |
| Region | ams (Amsterdam) — GDPR data residency |
| TLS | Automatic via Fly.io — HTTPS enforced |
| Screenshot service | Urlbox Starter plan (~€18/month) |
| Authentication | JWT (HS256) signed with shared secret |
| Rate limiting | 20 requests/minute per IP via slowapi |
| Monthly cost | ~€21–25 total (Fly.io + Urlbox) |

### Proxy codebase

Repository: `linklook-preview` (private, to be created).

```
linklook-preview/
├── main.py          — FastAPI app, routes, rate limiting
├── config.py        — Settings via pydantic-settings + .env
├── auth.py          — JWT bearer token verification
├── validator.py     — URL validation, private IP blocking (SSRF defense)
├── preview.py       — Urlbox API client, HMAC signing
├── requirements.txt
├── fly.toml         — Fly.io deployment config
└── .env             — Secrets (never commit)
```

**Known issue to fix before first deploy:** In `preview.py` the `hmac` import
creates a naming collision with the module. Fix: `import hmac as hmac_module`.

### Deployment

```bash
# 1. Authenticate
brew install flyctl && fly auth login

# 2. Create the app (first time only)
fly launch --name linklook-preview --region ams

# 3. Set secrets
fly secrets set URLBOX_API_KEY=xxx URLBOX_SECRET=xxx JWT_SECRET=xxx

# 4. Deploy
fly deploy
```

**Environment variables:**

| Variable | Notes |
|----------|-------|
| `URLBOX_API_KEY` | From Urlbox dashboard — set via `fly secrets` |
| `URLBOX_SECRET` | From Urlbox dashboard — set via `fly secrets` |
| `JWT_SECRET` | Generate with: `python -c "import secrets; print(secrets.token_hex(32))"` |

**Graceful degradation:** If the proxy is unreachable or Urlbox returns an error,
the iOS app falls back to showing only the verdict screen (no preview). Never
fail silently or show a blank preview pane. The proxy returns HTTP 502 on Urlbox
failure — the Swift client treats any non-200 as a fallback trigger.

### Privacy & GDPR

**Data processing summary:**

| Item | Detail |
|------|--------|
| Data collected | The URL submitted for preview |
| Association | Not linked to account, device ID, or any personal identifier |
| Retention | Zero — discarded immediately after screenshot is returned |
| Third-party processor | Urlbox (urlbox.io) — DPA required under Article 28 GDPR |
| Legal basis (EEA/UK) | Explicit consent (Article 6(1)(a) GDPR) |
| Default state | Off — user must opt in via onboarding consent screen |
| Withdrawal | Settings → Privacy → Safe preview — can be disabled at any time |

**Actions required before launch:**

1. Sign Urlbox Data Processing Agreement (available from Urlbox support).
2. Update LinkLook privacy policy with the approved copy below.
3. Implement onboarding consent screen using the approved copy below.
4. Verify server logs contain zero URL content (spot-check after first deploy).

### Approved copy

**Onboarding consent screen:**

| Element | Copy |
|---------|------|
| Title | Safe preview |
| Body | Before you open any link, LinkLook can show you a preview. To keep your device safe, the preview is loaded on our servers — not on yours. |
| Row 1 heading | Your device stays untouched |
| Row 1 body | The suspicious URL never loads on your iPhone. Our server fetches it instead. |
| Row 2 heading | URLs are not stored |
| Row 2 body | Links sent for preview are discarded immediately after analysis. We never log them. |
| Row 3 heading | One thing to know |
| Row 3 body | The destination site will see a request from LinkLook's server, not from you personally. |
| Legal footnote | By enabling safe preview you agree to URLs being sent to LinkLook's preview service. See our privacy policy for full details. |
| Primary CTA | Enable safe preview |
| Secondary CTA | Not now |

**Privacy policy section** (add as "Safe preview service"):

> **What it does.** When safe preview is enabled, LinkLook sends the URL of a
> link you are about to open to LinkLook's preview service. Our server fetches
> and renders the page on your behalf and returns a visual preview to your
> device. The suspicious URL never loads on your iPhone.
>
> **What we collect.** We receive the URL submitted for preview. We do not
> associate this URL with your account, your device identifier, or any other
> personal data. The URL is used solely to fetch the preview and is discarded
> immediately after the response is returned. We do not log, store, or analyse
> preview URLs beyond the duration of the request.
>
> **What the destination sees.** The destination website will receive a network
> request originating from LinkLook's server infrastructure, not from your
> device or your IP address. The destination cannot identify you from this
> request.
>
> **Third-party processor.** Preview rendering is performed by Urlbox
> (urlbox.io), acting as a data processor on our behalf under a Data Processing
> Agreement compliant with Article 28 GDPR. Urlbox processes the submitted URL
> only for the purpose of rendering the preview.
>
> **Legal basis (EEA/UK users).** Processing is based on your explicit consent,
> given at the time you enable safe preview (Article 6(1)(a) GDPR). You may
> withdraw consent at any time by disabling safe preview in Settings → Privacy.
> Withdrawal does not affect the lawfulness of processing prior to withdrawal.
>
> **Your control.** Safe preview is off by default. You can enable or disable
> it at any time in Settings → Privacy → Safe preview. When disabled, no URLs
> are sent to our servers; links open directly on your device.

### UX rules

1. **Only on INFORM and WARN** — OK doesn't need it (low risk), BLOCK doesn't
   show forward options anyway.
2. **User-initiated only** — Never auto-fetch the preview. The user taps
   "Preview page" explicitly.
3. **Loading state** — Show a spinner with "Generating preview…" (1–5 seconds
   typical).
4. **Timeout** — 8 seconds max. If the backend can't render in time, show
   "Preview unavailable" with a subtle retry option.
5. **Non-interactive** — The image is not tappable, not zoomable (v1). Pinch-to-
   zoom could be added later.
6. **Disclaimer** — Always show below the image: "Preview only — page not loaded
   on your device."
7. **Pro feature** — Safe Preview requires backend infrastructure and compute
   cost. It is a Pro feature.
8. **Consent required** — User must opt in via onboarding consent screen before
   first use. Off by default.

### Relationship to No-Preload Principle

Safe Preview does NOT violate the No-Preload Principle because:

- The page is never loaded on the user's device.
- The user's IP is never exposed to the destination.
- No code runs in the user's browser context.
- The preview is a flat image with no active content.
- The user explicitly requests the preview (not automatic).

Safe Preview extends the No-Preload Principle by giving the user visual
information about the destination while maintaining the safety guarantees.
The backend proxy acts as a shield between the user and the destination.

### Build plan

| Week | Work | Effort |
|------|------|--------|
| Week 1 | Server foundation: sign up Urlbox, test API with curl, scaffold FastAPI proxy, deploy to Fly.io EU, verify TLS and zero URL logging. | ~5 days |
| Week 2 | iOS integration: Swift client (submit URL, receive screenshot), preview pane in verdict screen, loading/timeout/error states, fallback. | ~5 days |
| Week 3 | Hardening: URL validation (private IP blocking), rate limiting, JWT auth, self-penetration test with Proxyman. | ~4 days |
| Week 4 | Privacy & legal: sign Urlbox DPA, update privacy policy, add onboarding consent screen. | ~2 days |
| Week 5 | Testing: XCUITest for preview flow, fallback, consent gate. Test with PhishTank URLs, malformed URLs, redirect chains. | ~4 days |

**Total estimated effort: 9–12 working days.**

### Future: self-hosted rendering (v2)

If Urlbox becomes a cost or control constraint, the rendering can be moved to
self-hosted Playwright in a disposable container on Fly.io Machines. The proxy
architecture stays the same — only the `preview.py` module changes from Urlbox
API calls to local Playwright rendering. The iOS client is unaffected.

### Contacts & accounts

| Item | Detail |
|------|--------|
| Product owner | Bas — all architectural and product decisions |
| Urlbox account | To be created — register at urlbox.io |
| Fly.io account | To be created or added as collaborator — fly.io |
| Apple Developer | Active membership held by Bas |
| Repo | To be created: `linklook-preview` (private) |
