# LinkLook — Privacy & Trust Roadmap

> **Date:** 2026-03-18
> **Status:** Living document — tracks privacy maturity from V1 to V3.

## NU (V1 — Required for App Store submission)

### 1. In-app consent bij cloud checks ✅ DONE
- CloudPrivacyConfirmationSheet in onboarding + Settings
- Only asks when privacy impact increases (upgrade)
- Never re-asks for same level or downgrades

### 2. Duidelijke tekst per niveau ✅ DONE
- Consistent across all surfaces (CLAUDE.md rule 70)
- On-device / Cloud link check / Cloud link & page check

### 3. Privacy Policy URL ✅ DONE (needs hosting)
- HTML page created: `LinkLook-Docs/080_compliance/legal/privacy-policy.html`
- Link in Settings → About (CLAUDE.md rule 69)
- **TODO:** Host at `https://linklook.app/privacy-policy`
- **TODO:** Enter URL in App Store Connect

### 4. App Privacy in App Store Connect 🔄 IN PROGRESS
- Guide created: `App_Store_Privacy_Guide.md`
- App created in App Store Connect (2026-03-20)
- Privacy Tracker spreadsheet created (App_Store_Privacy_Tracker.xlsx)
- **Decision:** No login in v1.0 → no Contact Info, no User ID, no Purchases
- **v1.0 datatypes:** Browsing History, Emails or Text Messages, Device ID, Product Interaction, Crash Data, Performance Data
- **All data:** Not linked to user, not used for tracking
- **TODO:** Complete questionnaire in App Store Connect (datatypes → purposes → publish)

### 5. Geen overdreven claims ✅ DONE
- Audit complete: no "fully private/secure/detects all" found
- CLAUDE.md rule 71 prevents future overclaiming

## V2 (Post-launch improvements)

### 1. Publieke privacy-uitlegpagina ✅ DONE (needs hosting)
- HTML page created: `privacy-choices.html`
- **TODO:** Host at `https://linklook.app/privacy-choices`

### 2. Settings privacy-sectie verbeteren ⬜ TODO
- Add: current protection level summary at top
- Add: "Turn off all cloud checks" quick button
- Add: link to "How LinkLook handles your data"

### 3. Logging/bewaarbeleid expliciet maken ⬜ TODO
- Document: do we log checked links? (server-side)
- Document: do we log page fetches?
- Document: retention periods per data type
- Document: what we explicitly do NOT store

### 4. Consistente privacy-teksten audit ⬜ TODO
- Verify identical descriptions across:
  onboarding / settings / website / privacy policy / support docs
- Run after every text change

## V3 (Trust & transparency)

### 1. Externe privacy/security review ⬜ FUTURE
- Independent code review
- Privacy review of data flows
- Security assessment of backend

### 2. Open source selectie ⬜ FUTURE
- Candidates: on-device checking logic, rule sets, consent flow
- NOT needed: backend, detection models

### 3. Publiek trust statement ⬜ FUTURE
- Short page: what's local, what's optional cloud, user control

### 4. Formele certificering ⬜ FUTURE
- Not needed for Apple review
- Relevant later for enterprise/partnerships

## Priority Order

| Phase | Action | Status |
|-------|--------|--------|
| **NU** | Consent flow | ✅ Done |
| **NU** | Privacy policy live | ⬜ Needs hosting |
| **NU** | App Store Privacy | 🔄 In progress |
| **NU** | Text consistency | ✅ Done |
| **NU** | No overclaiming | ✅ Done |
| **V2** | Public privacy page | ✅ Created, needs hosting |
| **V2** | Settings improvements | ⬜ |
| **V2** | Logging policy | ⬜ |
| **V3** | External review | ⬜ |
| **V3** | Open source | ⬜ |
| **V3** | Trust statement | ⬜ |
| **V3** | Certification | ⬜ |
