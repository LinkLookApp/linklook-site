# LinkLook — Privacy & Trust Roadmap

> **Date:** 2026-03-20
> **Status:** Living document — tracks privacy maturity from V1 to V3.

## NU (V1 — Required for App Store submission)

### 1. In-app consent bij cloud checks ✅ DONE
- CloudPrivacyConfirmationSheet in onboarding + Settings
- Only asks when privacy impact increases (upgrade)
- Never re-asks for same level or downgrades

### 2. Duidelijke tekst per niveau ✅ DONE
- Consistent across all surfaces (CLAUDE.md rule 70)
- On-device / Cloud link check / Cloud link & page check

### 3. Privacy Policy URL ✅ DONE
- HTML page created: `LinkLook-Docs/080_compliance/legal/privacy-policy.html`
- Link in Settings → About (CLAUDE.md rule 69)
- Hosted at `https://linklook.app/privacy-policy` (2026-03-20)
- URL ingevuld in App Store Connect (2026-03-20)

### 4. App Privacy in App Store Connect ✅ DONE
- Guide created: `App_Store_Privacy_Guide.md`
- App created in App Store Connect (2026-03-20)
- Privacy Tracker spreadsheet created (App_Store_Privacy_Tracker.xlsx)
- **Decision:** No login in v1.0 → no Contact Info, no User ID, no Purchases
- **v1.0 datatypes (definitief):** Browsing History, Emails or Text Messages, Device ID
- **Verwijderd:** Product Interaction, Crash Data, Performance Data (geen analytics/crash SDK geïnstalleerd)
- **All data:** Not linked to user, not used for tracking
- Privacy Policy URL ingevuld: `https://linklook.app/privacy-policy`
- Privacy Choices URL ingevuld: `https://linklook.app/privacy-choices`
- Privacy labels gepubliceerd (2026-03-20)

### 5. Geen overdreven claims ✅ DONE
- Audit complete: no "fully private/secure/detects all" found
- CLAUDE.md rule 71 prevents future overclaiming

### 6. Privacy review inhoud (2026-03-20) ✅ DONE
- GSB "partial URL hash" claim gecorrigeerd → "URL is sent to Google" in privacy-policy.html, privacy-choices.html, App_Store_Privacy_Guide.md
- "Emails or Text Messages" (Check Message Context) toegevoegd aan privacy-policy.html sectie 2
- Interne docs bijgewerkt (URL_Privacy_Signal_Spec.md, Test_Traceability_Matrix.txt)
- Bestanden gedeployed naar linklook.app (2026-03-20)

### 7. Nog open (NU) ⬜ TODO
- Juridische entiteit specificeren: "LinkLook is developed by LinkLook" → echte entiteit + adres
- Kinderen-leeftijd: "under 13" (COPPA) → "under 16" voor NL/EU GDPR

## V2 (Post-launch improvements)

### 1. Publieke privacy-uitlegpagina ✅ DONE
- HTML page created: `privacy-choices.html`
- Hosted at `https://linklook.app/privacy-choices` (2026-03-20)
- URL ingevuld in App Store Connect (2026-03-20)

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

### 5. Privacy-choices uitbreiden ⬜ TODO
- Safe Preview als apart niveau benoemen (nu vereenvoudigd naar 2 cloud levels, code heeft 3 toggles)

### 6. GDPR compliance verdiepen ⬜ TODO
- DPIA (Data Protection Impact Assessment) opstellen
- Verwerkingsregister (Article 30) formaliseren
- Subprocessors documenteren: Anthropic/Claude, Urlbox, Google (GSB/Web Risk)
- Internationale datatransfers documenteren: serverlocatie(s) + waarborgen

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

### 5. DPO-aanstelling ⬜ FUTURE
- Evalueren bij grote schaal

## Priority Order

| Phase | Action | Status |
|-------|--------|--------|
| **NU** | Consent flow | ✅ Done |
| **NU** | Privacy policy live | ✅ Done (hosted + ASC) |
| **NU** | App Store Privacy | ✅ Done |
| **NU** | Text consistency | ✅ Done |
| **NU** | No overclaiming | ✅ Done |
| **NU** | Privacy review inhoud | ✅ Done (GSB + Emails/Text fix) |
| **NU** | Juridische entiteit + kinderen-leeftijd | ⬜ |
| **V2** | Public privacy page | ✅ Done (hosted + ASC) |
| **V2** | Settings improvements | ⬜ |
| **V2** | Logging policy | ⬜ |
| **V2** | Privacy-choices uitbreiden | ⬜ |
| **V2** | GDPR compliance verdiepen | ⬜ |
| **V3** | External review | ⬜ |
| **V3** | Open source | ⬜ |
| **V3** | Trust statement | ⬜ |
| **V3** | Certification | ⬜ |
| **V3** | DPO-aanstelling | ⬜ |
