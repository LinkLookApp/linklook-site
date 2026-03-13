AI Test Fixtures — Readme
Status: active
Date: 2026-03-13

Purpose
This file explains the AI_Test_Fixtures.yaml format and how to use it.

File: AI_Test_Fixtures.yaml
Format: YAML with two sections: url_model_fixtures and message_model_fixtures.

URL model fixture fields:
  id          — test case ID (matches AI_Test_Catalog.txt)
  category    — true_positive, true_negative, suspicious, edge_case, etc.
  url         — the test URL (must use synthetic domains per CLAUDE.md rule 4)
  expected    — minimum/maximum score thresholds the model must meet
  features    — expected feature extractor outputs (for feature extractor tests)

Message model fixture fields:
  id          — test case ID
  category    — true_positive, true_negative, suspicious, multilingual, etc.
  text        — the message text to analyze
  channelHint — optional channel type (sms, email, chat, unknown)
  expected    — minimum/maximum score thresholds
  note        — optional explanation

Usage:
1. Feature extractor tests: parse url_model_fixtures, run URLFeatureExtractor,
   assert feature values match the "features" section.
2. Model accuracy tests: parse fixtures, run model, assert scores meet
   the "expected" thresholds.
3. Combined tests: use pairs of URL + message fixtures to verify that the
   message model never modifies the URL verdict.

Rules:
- All test domains must be clearly synthetic (CLAUDE.md rule 4).
- No real malicious URLs (CLAUDE.md rule 2).
- Fixture domains must NOT resolve to real sites (CLAUDE.md rule 3).
- New fixtures can be added; existing fixtures should not be weakened
  to match broken model behavior (CLAUDE.md rule 1).
