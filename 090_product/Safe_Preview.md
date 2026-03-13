# Safe Preview — Screenshot-Only Page Preview

**Status:** Future feature (requires backend)
**Date:** 2026-03-13
**Phase:** Post-launch, backend-dependent

## Problem

On the verdict screen, the user sees the domain name, full URL, verdict, and
signal badges — but not what the page actually looks like. For INFORM and WARN
verdicts, users often want to "peek" at the destination before deciding. Without
a preview, they must either trust the verdict blindly or proceed to the page.

Loading the page on the user's device to provide a preview would violate the
No-Preload Principle: it leaks the user's IP and intent, runs JavaScript, sets
cookies, and potentially triggers malicious behavior.

## Solution: Safe Preview

A backend service fetches and renders the destination page in a sandboxed
headless browser, captures a screenshot, and returns it as a flat image (PNG or
WebP). The user sees the page visually, but nothing runs on their device.

### How it works

1. User taps "Preview page" on the verdict screen (INFORM or WARN only).
2. The app sends the URL to the LinkLook backend.
3. The backend:
   - Fetches the page in a sandboxed headless browser (e.g. Puppeteer/Playwright
     in a disposable container).
   - Waits for initial render (2–3 seconds max).
   - Captures a viewport screenshot (mobile viewport, 390×844 or similar).
   - Strips EXIF/metadata from the image.
   - Returns the image to the app.
4. The app displays the screenshot as a non-interactive image below the
   destination card on the verdict screen.
5. The user can now visually assess the page without having loaded it.

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
│   Continue in LinkLook  Open in...   │
└──────────────────────────────────────┘
```

### Security properties

| Property | Guarantee |
|----------|-----------|
| No JavaScript on user device | The page runs only in the backend sandbox |
| No cookies set on user device | The user's browser state is untouched |
| No IP leak to destination | The backend's IP is exposed, not the user's |
| No tracking pixels fire on user device | All tracking executes in the sandbox only |
| No downloads on user device | The backend does not forward any downloads |
| Image is flat | PNG/WebP — no embedded scripts, no active content |
| Sandbox is disposable | Each render uses a fresh container, destroyed after |

### Privacy considerations

- The URL is sent to the LinkLook backend. This is already the case for backend
  checks. The privacy policy must disclose this.
- The backend fetches the page on behalf of the user. The destination server
  sees the backend's IP and user-agent, not the user's.
- The screenshot may contain sensitive content visible on the page (e.g. a login
  form, personal data if the page is publicly accessible). The app should display
  a disclaimer: "This is a preview image. The page has not been loaded on your
  device."
- Screenshots are not stored on the backend after delivery. Process, deliver,
  discard — same principle as context analysis.

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
   cost. It should be a Pro feature.

### Technical requirements (backend)

- Headless browser: Chromium via Puppeteer or Playwright.
- Container: Disposable (AWS Lambda + container, or Fly.io Machines).
- Viewport: Mobile (390×844), device-scale-factor 2 for retina.
- Timeout: 5 seconds for page load, 8 seconds total including render.
- Output: PNG or WebP, max 200KB (compress if needed).
- Rate limiting: per-user, to prevent abuse as a free screenshot service.
- No persistent storage: screenshot is delivered to the app and discarded.

### Relationship to No-Preload Principle

Safe Preview does NOT violate the No-Preload Principle because:

- The page is never loaded on the user's device.
- The user's IP is never exposed to the destination.
- No code runs in the user's browser context.
- The preview is a flat image with no active content.
- The user explicitly requests the preview (not automatic).

Safe Preview extends the No-Preload Principle by giving the user visual
information about the destination while maintaining the safety guarantees.
The backend acts as a proxy shield between the user and the destination.

### Implementation phases

1. **Backend endpoint:** `POST /api/v1/preview` — accepts URL, returns image.
2. **App integration:** "Preview page" button on INFORM/WARN verdict screens.
3. **Caching:** Cache previews for 5 minutes (same URL, same image) to reduce
   backend load on retry.
4. **Abuse prevention:** Rate limiting, URL validation, block internal/private
   IPs.
