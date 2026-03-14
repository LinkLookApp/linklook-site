# LinkLook — TestFlight Launch Plan

**Date:** 2026-03-13, 15:13 CET
**Device:** iPhone 11 (A13 chip, iOS 17+)
**Status:** 440 tests passing, core flow implemented

---

## What's blocking TestFlight right now

Based on a full audit of the project, here's what needs to happen — split into what Claude can do versus what you need to do yourself.

---

### A. You do (can't be done from Claude)

| # | Task | Time est. | Notes |
|---|------|-----------|-------|
| A1 | **Get Google Safe Browsing API key** | 10 min | Google Cloud Console → APIs & Services → Enable "Safe Browsing API" → Create API key. Paste into `Secrets.xcconfig` as `GSB_API_KEY = your-key-here` |
| A2 | **Create app icon (1024×1024 PNG)** | 15–60 min | The AppIcon asset catalog is empty — no image file. You need a 1024×1024 PNG. Options: design one yourself, use an AI image generator, or hire someone. Drop it into `LinkLook/Assets.xcassets/AppIcon.appiconset/` and update Contents.json with the filename. |
| A3 | **Real-device test (iPhone 11)** | 30 min | Run through TestFlight Readiness Guide Phase 2 checklist on the actual device. Simulator can't catch everything (camera for QR, paste permission prompts, browser redirect). |
| A4 | **App Store Connect setup** | 20 min | Create the app record in App Store Connect: app name "LinkLook", bundle ID `com.linklook.app`, subtitle "A safety-driven browser". Fill in privacy nutrition labels (URL hashes sent to Google for GSB). Add review notes (already drafted in `080_compliance/App_Review_Notes.txt`). |
| A5 | **Archive & upload from Xcode** | 10 min | Product → Archive → Distribute to App Store Connect → Upload. Then go to App Store Connect → TestFlight → select build → submit for review. |

### B. Claude does (code/config changes)

| # | Task | What changes |
|---|------|--------------|
| B1 | **Add PrivacyInfo.xcprivacy** | Apple requires this since 2024. Declares: network access (GSB API), camera (QR scanner). Add to both main app and Share Extension targets. |
| B2 | **Add export compliance flag** | `ITSAppUsesNonExemptEncryption = NO` in Info.plist. LinkLook only uses standard HTTPS (exempt). Prevents the manual compliance question on every upload. |
| B3 | **Add clipboard privacy description** | `NSPasteboardUsageDescription` in Info.plist — the app reads the clipboard for URL detection but lacks the required usage string. |
| B4 | **Add copyright string** | `NSHumanReadableCopyright` — currently empty. |
| B5 | **Fix TestFlight guide device reference** | Guide says "iPhone 16e" — should say "iPhone 11". |
| B6 | **Verify Debug UI is hidden in Release** | Already confirmed: `#if DEBUG` wraps the test tab. Just needs a final check that no debug print statements leak. |
| B7 | **Update App Review Notes** | Current draft is from ChatGPT and slightly outdated. Update to match current feature set (GSB integration, structural analysis, two-step safety flow). |

---

## What does NOT block TestFlight

Per the TestFlight Readiness Guide "stop perfecting" rule, these can ship in updates:

- AI URL model (Pro, not built yet)
- Context analysis (Pro, UI exists but AI not connected)
- Screenshot/image analysis (Pro, not built)
- Voice input
- Text size setting polish
- Early Access badges
- Perfect animations
- Edge case URL handling
- Backend reputation service

---

## Recommended sequence

1. **Claude does B1–B7** (~20 min)
2. **You do A1** (GSB key) — can happen in parallel
3. **You do A2** (app icon) — can happen in parallel
4. **Build + run all tests** to confirm green
5. **You do A3** (real-device test on iPhone 11, Phase 2 checklist)
6. **You do A4** (App Store Connect setup)
7. **You do A5** (Archive → Upload → Submit to TestFlight review)

Steps 1, 2, and 3 can all happen at the same time. The critical path is: icon + GSB key → build → device test → upload.
