# LinkLook — One-Page Policy Table

**Status:** Active — authoritative reference for the decision engine.
**Date:** 2026-03-13
**Canonical rules:** `CLAUDE.md` wins on any conflict with this document.

## Purpose

LinkLook classifies risk at click time. It does not claim to prove a zero-day
exploit. The goal is to help the user choose safely before a page is opened or
before a risky action is taken.

## Core Principle

> Unexpected + urgent + asks for login, payment, download, install, or remote
> access = escalate strongly.

---

## 1) Hard Policy Rules

These rules override the normal score.

| Rule | Result | Notes |
|------|--------|-------|
| Known malicious / phishing / malware reputation match | BLOCK | Highest-confidence reason |
| Clear phishing or malware behavior (fake login, forced download, fake update, fake support, install prompt, exploit-like behavior) | BLOCK | Behavior matters more than age alone |
| Exact URL is on a trusted allowlist and no conflicting strong negative signal exists | OK | Allowlist should be exact URL or tightly controlled domain/path |
| New domain alone | Never BLOCK by itself | "New" is a risk signal, not proof |

---

## 2) App Verdict Policy

| Verdict | Meaning | Typical signals | Default UI action | User options |
|---------|---------|-----------------|-------------------|--------------|
| **OK** | No immediate risk signals found | Expected context, normal or trusted domain, no sensitive request, no suspicious behavior | Show result as low-friction | Continue · Open in preferred browser |
| **INFORM** | Context unclear; user should apply judgment | Unexpected link, shortened or unfamiliar domain, QR link, newly registered / newly observed domain, low prevalence, limited trust context, but no strong scam signal yet | Show caution state | Go Back prominent; Continue · Open in preferred browser small |
| **WARN** | Meaningful scam or phishing indicators are present | Urgency, impersonation, login/payment reset request, invoice/package lure, misleading wording, suspicious redirects/pop-ups, new domain combined with other suspicious signals | Show strong warning state | Go Back prominent; Continue · Open in preferred browser very small |
| **BLOCK** | Dangerous or highly suspicious | Known malicious/phishing reputation, forced download, fake update, fake support alert, install prompt, exploit-like behavior, high-confidence phishing pattern | Do not proceed directly | Go Back only |

---

## 3) Signal-to-Verdict Mapping

| Signal | Default verdict | Notes |
|--------|-----------------|-------|
| Link was expected and fits the current task | OK | Lowers risk, but does not guarantee safety |
| Link is unexpected or out of context | INFORM | Context matters greatly |
| Shortened URL with unclear destination | INFORM | Raise further if combined with urgency or requests |
| Domain is unfamiliar but page is ordinary and asks for nothing sensitive | INFORM | Good case for user judgment |
| Newly registered / newly observed domain | INFORM | Useful signal, but not enough to block |
| Very low-prevalence domain | INFORM | Raise further if combined with other indicators |
| QR code from unknown or weak-trust source | INFORM | Raise further if destination is hidden or sensitive |
| Brand/domain mismatch or lookalike spelling | WARN | Strong phishing indicator |
| Urgent pressure ("act now", "account blocked", "invoice due today") | WARN | Escalate strongly |
| Requests for password, bank details, payment, codes, or identity data | WARN | If combined with impersonation, often BLOCK |
| Redirect chain or page behavior inconsistent with link context | WARN or BLOCK | Depends on severity |
| New domain + low prevalence + urgency / impersonation / sensitive request | WARN | This is where "new" becomes truly useful |
| Asks to install app, profile, certificate, extension, or remote access tool | BLOCK | High-risk action |
| Immediate download, fake update, fake virus alert, fake support screen | BLOCK | High-confidence dangerous pattern |
| Known bad URL / reputation match | BLOCK | Highest-confidence block reason |

---

## 4) Risk Scoring Model

Use scoring as a support layer, not as the only decision mechanism.
Hard rules above always win.

> **Calibration note (2026-03-13):** The numbers below are initial estimates.
> They will be calibrated with real-world telemetry from the feature vector
> logger once Phase 4 data is available. Do not treat them as validated
> thresholds.

### Base score

Start at 0.

### Risk additions

| Signal group | Score |
|-------------|-------|
| Domain registered 0–7 days ago | +20 |
| Domain registered 8–30 days ago | +12 |
| Newly observed by LinkLook / backend telemetry | +10 |
| Extremely low prevalence | +15 |
| Low prevalence | +8 |
| No reputation data | +5 |
| Weak / mixed negative reputation | +25 |
| Brand lookalike / typo domain | +20 |
| Suspicious subdomain stuffing | +15 |
| Raw IP URL | +20 |
| URL shortener | +10 |
| Excessive redirect chain | +10 |
| Sensitive login page on unusual domain | +30 |
| Download / install / profile / certificate / extension / remote-access prompt | +30 |
| Fake urgency / scare language | +15 |
| Aggressive permission prompts | +15 |
| Link arrived unexpectedly by SMS / WhatsApp / email / DM / ad / QR | +10 |
| User did not intend to visit this brand/site | +10 |

### Risk reductions

| Signal group | Score |
|-------------|-------|
| Trusted ongoing task / expected journey | −10 |
| Very common, long-observed domain | −8 |
| Strong allow signal from controlled exact URL allowlist | −15 |

---

## 5) Score-to-Verdict Thresholds

| Score | Verdict |
|-------|---------|
| 0–9 | OK |
| 10–24 | INFORM |
| 25–49 | WARN |
| 50+ | BLOCK |

### Guardrail

If the only signals are new domain and/or low prevalence, cap at INFORM unless
another meaningful suspicious signal appears.

---

## 6) Practical Interpretation

**OK** — The link fits what the user is doing, nothing sensitive is being asked,
and no suspicious reputation or behavior is visible.

**INFORM** — There is uncertainty, but not enough to say scam. Typical examples:
unfamiliar domain, QR code, new domain, shortened URL, weak context.

**WARN** — There are real phishing/scam indicators. Typical examples: urgency,
impersonation, package/invoice lure, suspicious redirect, new domain combined
with a sensitive request.

**BLOCK** — There is high-confidence danger. Typical examples: known bad
reputation, fake login impersonation, forced download, fake update, fake
support, install prompt, exploit-like behavior.

---

## 7) Product Rule in One Sentence

> Reputation and page behavior outweigh domain age; domain age and prevalence
> are supporting signals that increase caution but do not prove maliciousness
> on their own.
