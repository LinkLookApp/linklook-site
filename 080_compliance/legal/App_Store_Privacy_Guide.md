# LinkLook — App Store Privacy Questionnaire Guide

> **Purpose:** Reference for filling in App Store Connect → App Privacy.
> **Date:** 2026-03-18
> **Status:** Ready for App Store submission.

## 1. Data Collection — "Does your app collect data?"

**Yes** — but only when the user opts in to cloud features.

## 2. Data Types

### Browsing History
- **Collected:** Yes
- **Linked to user:** No
- **Used for tracking:** No
- **Purpose:** App Functionality
- **Notes:** Stored on-device only (up to 100 entries). Never sent to servers. User can clear in Settings.

### Other Data — URLs (when cloud features enabled)
- **Collected:** Yes (only with explicit user consent)
- **Linked to user:** No
- **Used for tracking:** No
- **Purpose:** App Functionality (security analysis)
- **Notes:** Only collected when user enables "Cloud URL Check" or "Cloud link & page check". Retained server-side up to 90 days.

## 3. Third-Party Partners

### Google Safe Browsing API
- **Data shared:** Partial URL hashes (not full URLs)
- **Purpose:** Checking URLs against known threat databases
- **Linked to user:** No
- **Used for tracking:** No

### No other third-party SDKs collect user data.

## 4. Privacy Nutrition Label Summary

| Data Type | Collected | Linked to Identity | Used for Tracking |
|-----------|-----------|-------------------|-------------------|
| Browsing History | Yes | No | No |
| Identifiers | No | — | — |
| Location | No | — | — |
| Contacts | No | — | — |
| User Content | No | — | — |
| Diagnostics (crash logs) | Only if iOS opt-in | No | No |

## 5. PrivacyInfo.xcprivacy (already configured)

```xml
NSPrivacyTracking: false
NSPrivacyCollectedDataTypes:
  - BrowsingHistory (AppFunctionality, not linked, not tracking)
NSPrivacyAccessedAPITypes:
  - UserDefaults (CA92.1)
```

## 6. Required URLs for App Store Connect

| Field | URL | Status |
|-------|-----|--------|
| Privacy Policy URL | `https://linklook.app/privacy-policy` | Needs hosting |
| Privacy Choices URL (optional) | `https://linklook.app/privacy-choices` | Needs hosting |

## 7. Hosting Checklist

- [ ] Host `privacy-policy.html` at `https://linklook.app/privacy-policy`
- [ ] Host `privacy-choices.html` at `https://linklook.app/privacy-choices`
- [ ] Enter Privacy Policy URL in App Store Connect → App Privacy
- [ ] Enter Privacy Choices URL in App Store Connect → App Privacy (optional)
- [ ] Add Privacy Policy link in app Settings (see CLAUDE.md rule 69)
