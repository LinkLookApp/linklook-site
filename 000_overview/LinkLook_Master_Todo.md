# LinkLook — Master Todo List

> **Last updated:** 2026-03-19
> **Status:** Living document — update after every work session.

---

## NU — Privacy & App Store Submission

- [ ] App Store Connect Privacy questionnaire invullen (guide: `080_compliance/legal/App_Store_Privacy_Guide.md`)

## NU — Apple Review Readiness

- [ ] Strengthen "real browser" story
- [ ] Prepare clear App Review notes
- [ ] Make App Store privacy details fully accurate
- [ ] Geef één reproduceerbare testflow in review notes

## NU — Gap Analysis

- [ ] Hybrid sensitive data detection: check of het in code, docs én tests zit
- [ ] Check which tests are missing (code vs tests, docs vs tests)
- [ ] Check which docs are missing
- [ ] Which test types are missing?
- [ ] Judge all browser pages on Safari-likeness (screenshots → review)

## NU — Build & Bugs

- [ ] Build fixen en valideren (Hashable/Identifiable fix doorgevoerd, nog niet gevalideerd)
- [ ] Tab switching testen op device (ForEach+ZStack aanpak verifiëren)
- [ ] Ontbrekende overflow-tests: Favorites (50), ReadLater (200), Downloads (100), FalsePositiveReportStore (50), GSBCache (500)
- [ ] Timeout-test gaps: "Open" knop op timeout-scherm, exacte timeout-tekst, gelijktijdige timeouts

---

## TestFlight

- [ ] Focused test pass iPhone + iPad
- [ ] Release-mode regression pass
- [ ] Check beta metadata before TestFlight submission
- [ ] Deploy on TestFlight

## Entitlement & Default Browser

- [ ] Make UI entitlement ready (zie ChatGPT discussie)
- [ ] Inhoud privacy-policy.html en privacy-choices.html reviewen vóór entitlement request
- [ ] Fill entitlement form
- [ ] Set Default app help (zie ChatGPT chat 16 maart)
- [ ] Test < 18.2 versions op "Default Browser" handling na entitlement
- [ ] Adjust tests on default browser after Network Extension implementation

## Post-Entitlement

- [ ] Check external links

---

## Backend

- [ ] How does current version handle whitelist on frontend?
- [ ] Send GSB result to backend
- [ ] Send iOS version to backend (check if already in logs)
- [ ] Heeft Claude backend instructies nodig? Hoe doe je dat?
- [ ] Verdict frontend verwerken in backend after API Claude
- [ ] Sanitize and training data — do we need to sanitize?
- [ ] Implement black/greylist (zie ChatGPT)
- [ ] Verdict frontend verwerken in B/G-lists
- [ ] Zero day check
- [ ] Can we verify claims via logs?

## Backend — Business

- [ ] Pricing model — more than 2 layers?

---

## Testing

- [ ] Safari extensions testen
- [ ] Crawl links + mass link test after crawl
- [ ] Adjust default browser tests after NE implementation

---

## Business & Partnerships

- [ ] SWOT analyse
- [ ] Contact Henriette Bongers (directeur Fraudehelpdesk) — potentiële partner/adviseur. LinkedIn post (maart 2026) over UNODC-conferentie online fraude: nadruk op netwerksamenwerking, preventie, slachtofferperspectief. Relevant voor LinkLook: "It takes a network to fight a (criminal) network." Mogelijke samenwerking op meldingen, preventie-data, vertrouwenssignaal.

---

## V2 — Post-launch improvements

- [ ] Settings privacy-sectie verbeteren (samenvatting huidige stand, directe uit-knop, links)
- [ ] Logging/bewaarbeleid expliciet documenteren
- [ ] Consistente privacy-teksten audit (alle surfaces)

## V3 — Trust & Transparency

- [ ] Externe privacy/security review
- [ ] Open source selectie (on-device logic, rules, consent flow)
- [ ] Publiek trust statement pagina
- [ ] Formele certificering evalueren

---

## Afgerond (2026-03-19)

- [x] Tab overview thumbnails — captureAllSnapshots()
- [x] Tab bar hideable — floating LinkLook logo als toggle
- [x] Tab switching fix — ForEach+ZStack, strong WKWebView reference
- [x] Tabs-knop uniform rechtsonder (Safari-stijl)
- [x] QR-scanner naar floating bottom bar + browser toolbar
- [x] Eenmalige tooltips ("Menu" / "Scan QR")
- [x] Cloud privacy confirmation sheets (onboarding + Settings)
- [x] MinimalBrowserView.swift → BrowserCoordinator / WebViewRepresentable / ShareSheet
- [x] Dode code opgeruimd (cardStyle, angry.gif, warn.gif)
- [x] Tijdelijke scripts verwijderd, structurele naar scripts/
- [x] Privacy Policy + Privacy Choices HTML
- [x] App Store Privacy Guide
- [x] Privacy links in Settings
- [x] Alle tests bijgewerkt + nieuwe tests
- [x] CLAUDE.md regels 63–71
- [x] Start_Page_Design.txt v4.0
- [x] Cloud_Analysis_Consent_Spec.md uitgebreid
- [x] Capacity Limits tabel in CLAUDE.md
- [x] Privacy teksten gelijktrekken (naming consistency)
- [x] Overclaiming audit
- [x] Privacy_Trust_Roadmap.md
- [x] Script naming convention (rule in CLAUDE.md)
- [x] "After every code change → check tests and docs" regel
- [x] Stale referenties MinimalBrowserView opgeruimd
- [x] Host `privacy-policy.html` op `https://linklook.app/privacy-policy`
- [x] Host `privacy-choices.html` op `https://linklook.app/privacy-choices`
