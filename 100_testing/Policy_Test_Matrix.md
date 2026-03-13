# LinkLook — Policy Test Matrix

**Status:** Active — spec for test suite implementation.
**Date:** 2026-03-13
**Policy source:** `LinkLook-Docs/060_security/Policy_Table.md`

This matrix turns the LinkLook policy into concrete, testable cases.

**Important:** The URLs below are illustrative test fixtures, not live malicious
sites. They use IANA-reserved `.example` domains so they can safely live in
docs and tests.

## Column key

| Column | Meaning |
|--------|---------|
| Rule ID | Stable test case identifier |
| Rule / signal | The policy condition being tested |
| User message / context | How the link reaches the user |
| Example URL | Fixture URL for tests |
| Observed page behavior | What LinkLook or the user would observe |
| Expected verdict | OK / INFORM / WARN / BLOCK |
| Why | Short justification |
| **v1 detectable?** | **Whether the current v1 engine can detect this signal** |

### v1 signal sources

| Source | Available in v1? | Notes |
|--------|-----------------|-------|
| **Structural** | Yes | URL pattern analysis, TLD, IP detection, brand lookalike, subdomain stuffing, entropy |
| **GSB** | Yes (with API key) | Binary: threat match or clean |
| **On-device AI** | Pro only (Phase 4) | Core ML model on URL features |
| **Page analysis** | No — future | Requires loading and inspecting page content/DOM |
| **Reputation DB** | No — future | Requires backend with crawl/telemetry data |
| **Context analysis** | Pro only (Phase 4) | Analyzes surrounding message text, not the URL |

---

## A. Hard Policy Override Tests

| Rule ID | Rule / signal | User message / context | Example URL | Observed page behavior | Expected verdict | Why | v1 detectable? |
|---------|---------------|----------------------|-------------|----------------------|-----------------|-----|----------------|
| HP-01 | Known malicious reputation match | SMS: "Your parcel is waiting. Confirm now." | `https://track-confirm.bad-reputation.example/parcel` | Threat intel / reputation service returns known phishing hit | BLOCK | Hard override: known bad URL/domain | **GSB only** — depends on GSB having this URL. No own reputation DB yet. |
| HP-02 | Known malware reputation match | Email: "Open attached invoice portal" | `https://invoice-dropper.bad-reputation.example/open` | Reputation marks URL as malware delivery | BLOCK | Hard override: known malware delivery | **GSB only** |
| HP-03 | Clear phishing login impersonation | Email appears to be from bank | `https://rabobank-login-security.example/signin` | Fake bank login on non-bank domain | BLOCK | Hard override: high-confidence phishing behavior | **Structural: yes** — brand lookalike + login path detectable. Page content analysis: no. |
| HP-04 | Forced download behavior | Pop-up ad: "Open protected document" | `https://docs-viewer.example/download` | Immediate download starts without user intent | BLOCK | Hard override: dangerous download behavior | **No** — requires page analysis. Structural can flag `/download` path as suspicious (WARN at best). |
| HP-05 | Fake update prompt | Browser pop-up: "Your iPhone is out of date" | `https://ios-update-now.example/update` | Page pushes fake system update flow | BLOCK | Hard override: fake update pattern | **Partial** — structural can detect brand impersonation ("ios" in non-Apple domain) + suspicious path. Full detection requires page analysis. |
| HP-06 | Fake support / scare screen | Search result clicked by accident | `https://apple-support-alert.example/help` | Full-screen scare text, alarm, "call support now" | BLOCK | Hard override: fake support pattern | **Partial** — structural can detect "apple-support" brand impersonation. Scare behavior requires page analysis. |
| HP-07 | Install profile / certificate prompt | WhatsApp: "Install secure work access" | `https://secure-access-profile.example/install` | Prompts user to install configuration profile / certificate | BLOCK | Hard override: install/profile action | **No** — requires page analysis. Structural can flag `/install` path. |
| HP-08 | Exact allowlist hit with no conflict | User taps known internal portal bookmark | `https://portal.example.com/payroll` | Exact URL on trusted allowlist, no suspicious signs | OK | Hard override: allow trusted exact URL | **Yes** — allowlist is structural check. |
| HP-09 | New domain only | User receives plain marketing link | `https://newbrand-launch.example/offer` | Domain 3 days old, ordinary page, nothing sensitive | INFORM | Guardrail: new alone must not block | **Partial** — domain age requires WHOIS/backend lookup (not in v1). Structural analysis may flag unfamiliar TLD or patterns. |

---

## B. Base Signal-to-Verdict Tests

| Rule ID | Rule / signal | User message / context | Example URL | Observed page behavior | Expected verdict | Why | v1 detectable? |
|---------|---------------|----------------------|-------------|----------------------|-----------------|-----|----------------|
| SV-01 | Expected link fits current task | User in known webshop checkout flow | `https://shop.example.com/checkout/payment` | Domain matches ongoing task, no suspicious prompts | OK | Expected context lowers risk | **Partial** — structural sees clean domain. "Expected task" context not available to engine. |
| SV-02 | Unexpected link out of context | SMS from unknown sender: "Look at this" | `https://photoshare.example/album/9281` | Ordinary page, no sensitive ask | INFORM | Unexpected context requires caution | **Structural: yes** — unfamiliar domain detectable. "Unknown sender" context requires context analysis (Phase 4). |
| SV-03 | Shortened URL, unclear destination | SMS: "Please confirm here" | `https://short.example/a8K2pQ` | Destination obscured before expansion | INFORM | Destination unclear | **Structural: yes** — URL shortener pattern detectable. |
| SV-04 | Unfamiliar domain, ordinary page | User sees ad and taps link | `https://green-garden-tools.example/spring` | Simple marketing page, no login/payment/download | INFORM | Unfamiliar but non-sensitive | **Structural: yes** — unfamiliar domain flagged. |
| SV-05 | Newly registered domain | Email newsletter from unknown source | `https://freshbrand.example/welcome` | Domain age 5 days, otherwise ordinary | INFORM | Newness raises caution only | **No** — domain age requires WHOIS lookup (future). Structural may flag as unfamiliar. |
| SV-06 | Very low-prevalence domain | Link posted in a new chat group | `https://rare-offer.example/info` | Little/no prior telemetry, no strong bad signal | INFORM | Low prevalence is supporting signal | **No** — prevalence requires backend telemetry (future). |
| SV-07 | QR code from weak-trust source | Printed flyer with QR code in public place | `https://qr-landing.example/promo` | Destination hidden until scanned; page ordinary | INFORM | Weak-trust QR context | **Partial** — engine knows URL came from QR scanner entry point. Domain analysis is structural. |
| SV-08 | Brand/domain mismatch | SMS: "ING security check" | `https://ing-verification-center.example/login` | Brand name in domain, domain not owned by brand | WARN | Strong phishing indicator | **Structural: yes** — brand lookalike detection. |
| SV-09 | Lookalike spelling | Email: "Microsoft document shared with you" | `https://micr0soft-docs.example/share` | Typosquatted / deceptive brand lookalike | WARN | Strong phishing indicator | **Structural: yes** — typosquatting / character substitution detection. |
| SV-10 | Urgency pressure | SMS: "Act now. Your account will be closed today." | `https://account-review.example/urgent` | Strong time pressure language | WARN | Urgency is a major scam signal | **No** — urgency is in message text, not URL. Requires context analysis (Phase 4). Structural can flag suspicious path/domain. |
| SV-11 | Sensitive credential request | Email: "Re-enter password to keep mailbox active" | `https://mail-restore.example/signin` | Requests username, password, MFA code | WARN | Sensitive ask; elevate strongly | **Partial** — structural detects `/signin` path + unfamiliar domain. Actual credential form requires page analysis. |
| SV-12 | Payment request | SMS: "Unpaid toll invoice due today" | `https://pay-toll-now.example/pay` | Asks for card details and billing address | WARN | Sensitive payment request | **Partial** — structural flags `/pay` path + suspicious domain. Actual payment form requires page analysis. |
| SV-13 | Redirect chain inconsistent with context | User thinks they're opening a receipt | `https://receipt-view.example/r/9812` | Link redirects through 4 unrelated domains | WARN | Suspicious redirect behavior | **No** — redirect chain following requires HEAD requests (future). |
| SV-14 | New domain + urgency + sensitive request | SMS: "Package held. Pay €1.95 within 1 hour" | `https://postnl-fee-check.example/pay` | Domain 2 days old, low prevalence, asks for payment | WARN | Combined scam indicators | **Partial** — structural: brand impersonation ("postnl") + `/pay` path. Domain age and message urgency not available in v1. |
| SV-15 | Install app / extension / remote tool | Fake IT helpdesk message | `https://helpdesk-assist.example/connect` | Asks to install remote support app | BLOCK | High-risk action | **No** — requires page analysis. Structural flags suspicious domain only. |
| SV-16 | Immediate download / fake virus alert | Search result opens alarming page | `https://device-infected.example/scan` | Fake virus warning with forced download CTA | BLOCK | High-confidence dangerous pattern | **Partial** — structural flags alarming domain name. Page behavior requires page analysis. |
| SV-17 | Known bad URL / reputation match | WhatsApp: "Track your package here" | `https://parcel-check.bad-reputation.example/track` | Reputation service says phishing | BLOCK | Highest-confidence block | **GSB only** |

---

## C. Scoring and Guardrail Tests

| Rule ID | Rule / signal | User message / context | Example URL | Observed page behavior | Expected verdict | Why | v1 detectable? |
|---------|---------------|----------------------|-------------|----------------------|-----------------|-----|----------------|
| SC-01 | Score stays low | User opens known news article | `https://news.example.com/article/2026/03/13/story` | Long-observed domain, expected context, no risky ask | OK | Should remain below INFORM threshold | **Structural: yes** — clean common domain. |
| SC-02 | New domain only guardrail | User receives generic promo link | `https://brandnew-campaign.example/sale` | Domain 4 days old, low prevalence, no sensitive action | INFORM | Guardrail prevents WARN/BLOCK from age alone | **Partial** — domain age needs WHOIS. Structural sees unfamiliar domain. |
| SC-03 | Low prevalence only guardrail | User opens niche hobby site | `https://rare-model-trains.example/forum` | Rare domain, but content ordinary | INFORM | Low prevalence alone should not over-escalate | **No** — prevalence requires backend telemetry. |
| SC-04 | Multiple medium-risk signals combine to WARN | Unknown sender shares "invoice" | `https://billing-review.example/doc/8821` | New domain, low prevalence, shortened redirect, urgency text | WARN | Combined signals cross warn threshold | **Partial** — structural flags unfamiliar domain + suspicious path. Full signal combination requires domain age + prevalence + message context. |
| SC-05 | Reputation weak negative | User clicks ad result | `https://coupon-hub.example/deal` | Weak/mixed bad reports, aggressive pop-ups, no credential ask | WARN | Elevated caution on weak intel | **No** — requires reputation database (future). GSB is binary, not graduated. |
| SC-06 | Sensitive login on unusual domain → BLOCK | Email: "Your Microsoft 365 session expired" | `https://m365-session-reset.example/signin` | Unusual domain + fake enterprise login | BLOCK | High-confidence phishing pattern | **Structural: yes** — brand impersonation ("m365") + `/signin` path. Strong signal combination. |
| SC-07 | Old domain but malicious behavior | Link shared in forum | `https://legacy-site.example/update/player` | Domain years old but runs fake update flow | BLOCK | Old age does not imply safe | **No** — requires page analysis. Structural sees clean-looking old domain. GSB may catch it. |
| SC-08 | Expected journey but dangerous behavior | Shopping email checkout link | `https://shop-confirm.example/checkout` | Seems expected, but page asks to install payment plugin | BLOCK | Dangerous behavior overrides expected context | **No** — requires page analysis. |

---

## D. Message Pattern Tests

> **Note:** These tests validate message context signals. In v1, the surrounding
> message text is NOT available to the URL check engine. These tests apply to
> the **Context Analysis** feature (Pro, Phase 4). The URL structural analysis
> may still produce a partial verdict based on the URL alone.

| Rule ID | Rule / signal | User message / context | Example URL | Observed page behavior | Expected verdict | Why | v1 detectable? |
|---------|---------------|----------------------|-------------|----------------------|-----------------|-----|----------------|
| MP-01 | Package lure | "Your package could not be delivered. Confirm address now." | `https://parcel-fix.example/address` | Payment/address confirmation flow | WARN | Common scam lure with sensitive request | **Context analysis: Phase 4.** Structural: unfamiliar domain only (INFORM). |
| MP-02 | Invoice lure | "Invoice #88431 overdue. Pay today to avoid penalty." | `https://invoice-review.example/pay` | Payment page on unfamiliar domain | WARN | Urgency + money request | **Context analysis: Phase 4.** Structural: `/pay` path flag (INFORM/WARN). |
| MP-03 | Password reset lure | "Your mailbox will be disabled. Verify your password." | `https://mail-verify.example/login` | Credential request | WARN | Sensitive credential lure | **Context analysis: Phase 4.** Structural: `/login` + unfamiliar domain (WARN possible). |
| MP-04 | Gift / prize lure | "You won a phone. Claim in the next 10 minutes." | `https://claim-prize.example/reward` | Countdown timer, data collection form | WARN | Pressure + lure | **Context analysis: Phase 4.** Structural: suspicious domain only. |
| MP-05 | Tech support lure | "Virus detected. Call Apple Support immediately." | `https://support-emergency.example/alert` | Full-screen scare page | BLOCK | Classic fake support pattern | **Context analysis: Phase 4.** Structural: brand impersonation possible (WARN). |
| MP-06 | Work access lure | "Install this secure profile to read the document." | `https://document-access.example/install` | Profile / certificate install prompt | BLOCK | High-risk installation request | **Context analysis: Phase 4.** Structural: `/install` path flag. |
| MP-07 | Innocent social share | "Here are the photos from yesterday" from known friend | `https://photos.example.com/shared/abc123` | Normal media page, no login needed | OK | Expected context and low-risk behavior | **Structural: yes** — clean domain, no flags. |
| MP-08 | Ambiguous social share from unknown sender | "See this now" | `https://photoshare.example/view/993` | Ordinary page, unknown sender, no sensitive ask | INFORM | Unexpected but not strongly malicious | **Context analysis: Phase 4.** Structural: unfamiliar domain (INFORM). |

---

## E. URL Fixture Categories for Automated Tests

Use these fixture categories in test code so the policy remains stable even if
exact strings change.

| Fixture category | Example URL | Intended meaning |
|-----------------|-------------|------------------|
| `trusted_expected_url` | `https://shop.example.com/checkout/payment` | Known good, expected journey |
| `new_but_ordinary_url` | `https://freshbrand.example/welcome` | New domain only |
| `shortener_url` | `https://short.example/a8K2pQ` | Hidden destination |
| `lookalike_brand_url` | `https://micr0soft-docs.example/share` | Brand impersonation |
| `sensitive_login_url` | `https://mail-restore.example/signin` | Credential request |
| `payment_lure_url` | `https://pay-toll-now.example/pay` | Payment request |
| `forced_download_url` | `https://docs-viewer.example/download` | Immediate download |
| `fake_update_url` | `https://ios-update-now.example/update` | Fake update flow |
| `profile_install_url` | `https://secure-access-profile.example/install` | Profile/certificate install |
| `known_bad_url` | `https://track-confirm.bad-reputation.example/parcel` | Reputation block |

---

## F. Minimal Acceptance Criteria

For this matrix to pass:

1. New domain alone must never return BLOCK.
2. Known bad reputation must always return BLOCK.
3. Install / profile / certificate / remote access prompts must always return BLOCK.
4. Urgency + impersonation + sensitive request must return at least WARN, often BLOCK.
5. Expected, ordinary, non-sensitive links must return OK.
6. Unexpected but ordinary links must usually return INFORM.
7. Old domain with dangerous behavior must still return BLOCK.

---

## G. Suggested Test Suite Split

| Suite | Cases | v1 implementable? |
|-------|-------|-------------------|
| `PolicyOverrideTests` | HP-01 to HP-09 | Mostly yes (structural + GSB). HP-04, HP-07 need page analysis stubs. |
| `SignalVerdictMappingTests` | SV-01 to SV-17 | Partially — structural signals yes, domain age/prevalence/page behavior need stubs or future work. |
| `ScoringGuardrailTests` | SC-01 to SC-08 | Partially — score model needs domain age + prevalence inputs that aren't available yet. |
| `MessagePatternTests` | MP-01 to MP-08 | Phase 4 (context analysis). Can stub with expected structural-only verdicts for now. |
| `FixtureCatalogTests` | Fixture validity | Yes — verify fixture URLs use `.example` TLD and don't resolve. |
