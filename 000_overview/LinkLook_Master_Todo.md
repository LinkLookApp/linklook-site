# LinkLook — Master Todo List

> **Last updated:** 2026-03-20
> **Status:** Living document — update after every work session.

---

## NU — Privacy & App Store Submission

- [x] App Store Connect Privacy questionnaire invullen — KLAAR
- [x] **KRITIEK:** Privacy policy + choices + App Store Guide: GSB "partial URL hash" → gecorrigeerd naar "URL is sent to Google" (2026-03-20)
- [x] **KRITIEK:** "Emails or Text Messages" (Check Message Context) toegevoegd aan privacy-policy.html sectie 2 (2026-03-20)
- [x] Privacy-choices.html: GSB-tekst gecorrigeerd (2026-03-20)
- [x] App_Store_Privacy_Guide.md: bijgewerkt — GSB-claim, hosting status, Web Risk notitie (2026-03-20)
- [x] Privacy_Trust_Roadmap.md: statussen bijgewerkt — hosting done, privacy review sectie toegevoegd, GDPR items bij V2 (2026-03-20)
- [ ] Kinderen-leeftijd in privacy policy: "under 13" (COPPA) → "under 16" voor NL/EU GDPR compliance
- [ ] Privacy policy sectie 1 bijwerken zodra BV is opgericht (naam + KvK + adres)

## PARALLEL — BV-oprichting (niet blokkend voor launch)

> **Beslissing 2026-03-20:** LinkLook B.V. oprichten voor aansprakelijkheidsbescherming. Loopt parallel aan development — niet blokkend voor entitlement of App Store submission. Privacy policy wordt bijgewerkt zodra KvK-nummer beschikbaar is.

- [x] **Stap 1:** Notaris gekozen: UwBVoprichten.nl (Damsté Notarissen), aanvraag ingediend €399, normaal traject, verwacht ~7 april 2026
- [ ] **Stap 2:** Statuten laten opstellen (notaris doet dit, keuzes: naam "LinkLook B.V.", doel, aandelenstructuur — standaard 1 aandeelhouder/bestuurder is prima)
- [ ] **Stap 3:** Bankrekening openen op naam van B.V. i.o. (minimaal €0,01 stortingskapitaal)
- [ ] **Stap 4:** Oprichtingsakte ondertekenen bij notaris → KvK-inschrijving volgt automatisch
- [ ] **Stap 5:** Na KvK: privacy policy updaten met BV-naam + KvK-nummer + deployen, Apple Developer account overzetten naar BV
- [ ] **Stap 6 (post-launch):** Vestigingsadres omzetten van privéadres naar virtueel kantooradres (~€40/maand) → KvK wijzigen + privacy policy updaten

## NU — Analytics & Crash SDK beslissing

- [ ] Beslissen of analytics/crash SDK (Firebase, Crashlytics, Sentry, etc.) nodig is vóór v1.0 release
- [ ] Zo ja: integreren + App Store Privacy labels updaten (Product Interaction, Crash Data, Performance Data toevoegen)

## NU — Google Web Risk Migratie (vóór App Store)

- [ ] Google Cloud account aanmaken + Web Risk API enablen
- [ ] Web Risk API key genereren → `Secrets.xcconfig` als `WEB_RISK_API_KEY`
- [ ] GSBLookupClient refactoren naar WebRiskClient (Web Risk endpoint + auth)
- [ ] Dual-config: Safe Browsing v4 voor dev/TestFlight, Web Risk voor Release
- [ ] Smoke test Web Risk integratie (clean URL + known-bad URL)

> **Beslissing 2026-03-20:** Safe Browsing v4 blijft voor development/TestFlight (gratis, geen ToS-issue). Migratie naar Web Risk API vóór App Store lancering — dat is de commercieel conforme route. 100.000 gratis calls/maand dekt ruim tot schaling.

## NU — Apple Review Readiness

- [ ] Strengthen "real browser" story
- [ ] Prepare clear App Review notes
- [ ] Make App Store privacy details fully accurate (incl. GSB + Emails/Text fixes hierboven)
- [ ] Geef één reproduceerbare testflow in review notes

## NU — Gap Analysis

- [ ] Hybrid sensitive data detection: check of het in code, docs én tests zit
- [ ] Check which tests are missing (code vs tests, docs vs tests)
- [ ] Check which docs are missing
- [ ] Which test types are missing?
- [ ] Judge all browser pages on Safari-likeness (screenshots → review)
- [ ] Privacy-specifieke tests bouwen:
  - [ ] Canary log test: inject fake sensitive data → verifieer niet in logs (staat in Consent Spec als planned)
  - [ ] Consent scope test: verifieer dat `/analyze-url` nooit page content fetcht
  - [ ] Data retention test: verifieer auto-purge na 90 dagen in audit logger
  - [ ] Privacy Signal test: verifieer masking vóór elke backend call
  - [ ] Tekst-consistentie test: geautomatiseerde check onboarding ↔ settings ↔ website ↔ privacy policy

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
- [x] Inhoud privacy-policy.html en privacy-choices.html reviewen vóór entitlement request — KLAAR (review 2026-03-20, fixes in NU-sectie)
- [ ] Privacy HTML's finaliseren vóór entitlement:
  - [ ] Sectie 1: juridische entiteit invullen (zodra BV rond is, of tijdelijke formulering)
  - [ ] Sectie 9: kinderen-leeftijd "under 13" → "under 16" (NL/EU GDPR)
  - [ ] Effective date bijwerken naar datum van laatste wijziging
  - [ ] Laatste leescheck: toon, consistentie, taalfouten
  - [ ] Deploy naar linklook.app
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
- [ ] Sign in with Apple implementeren (bij premium release)
- [ ] Subscription/IAP opzetten
- [ ] App Store Privacy labels updaten bij premium (Contact Info, User ID, Purchases toevoegen)

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
- [ ] Privacy-choices.html uitbreiden: Safe Preview als apart niveau benoemen (nu vereenvoudigd naar 2 cloud levels, code heeft 3 toggles)
- [ ] DPIA (Data Protection Impact Assessment) opstellen — sterk aanbevolen vóór cloud features live gaan
- [ ] GDPR verwerkingsregister (Article 30) formaliseren — veel info staat al in docs, nog niet in officieel formaat
- [ ] Subprocessors documenteren: Anthropic/Claude (page analysis), Urlbox (screenshots), Google (GSB/Web Risk)
- [ ] Internationale datatransfers documenteren: serverlocatie(s) + waarborgen (SCCs indien buiten EU)

## V3 — Trust & Transparency

- [ ] Externe privacy/security review
- [ ] Open source selectie (on-device logic, rules, consent flow)
- [ ] Publiek trust statement pagina
- [ ] Formele certificering evalueren
- [ ] DPO-aanstelling evalueren (bij grote schaal)

---

## Beslissingen (2026-03-20)

- **Geen login in v1.0** — app blijft laagdrempelig, geen account nodig. Scan-geschiedenis lokaal op device.
- **Sign in with Apple bij premium/v2.0** — pas wanneer sync, subscriptions of persoonlijke alerts komen.
- **App Store Privacy v1.0 datatypes (definitief):** Browsing History, Emails or Text Messages, Device ID. Alle drie: not linked to user, no tracking, App Functionality.
- **Product Interaction, Crash Data, Performance Data verwijderd** — geen analytics/crash SDK geïnstalleerd, dus niet declareren. Toevoegen zodra Firebase/Crashlytics/Sentry wordt geïntegreerd.
- **Emails or Text Messages:** nodig vanwege context-analyse feature (gebruiker plakt tekst uit e-mail/bericht zodat AI de link in context beoordeelt).
- **Device ID bron:** pseudonieme ID uit AnalysisAuditLogger (niet IDFA, niet persistent na herinstallatie). Gebruikt voor audit logging bij cloud analyse-requests.
- **Foundation claims alleen bij premium** — verificatie via Sign in with Apple, niet via los e-mailadres. Device ID volstaat niet voor claims; Apple ID geeft sterkere identiteitsverificatie.

## Afgerond (2026-03-20)

- [x] CI sync prompts aangescherpt: compilatie/test verboden in micro-sync en test-sync (MDL-120) — ~50-70 min besparing per CI run
- [x] App aangemaakt in App Store Connect (LinkLook, com.linklook.app)
- [x] Privacy Tracker spreadsheet (App_Store_Privacy_Tracker.xlsx)
- [x] App Store Connect Privacy questionnaire volledig ingevuld en gepubliceerd
  - Datatypes: Browsing History, Emails or Text Messages, Device ID
  - Privacy Policy URL + Privacy Choices URL ingevuld
  - Privacy labels gepubliceerd

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
