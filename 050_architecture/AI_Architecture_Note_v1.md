# LinkLook AI Architecture Note v1

## Goal
Add AI in a controlled way as an extra signal layer for LinkLook.
AI must not replace rules, policy, or backend reputation. AI is additive.

## Scope of the first AI phase

Two separate AI tasks:

1. **Link signal model**
   - Purpose: give an extra signal for the link itself
   - Input: checked URL string and derived URL features
   - Output: risk score + class probabilities
   - Role: support verdicting, never decide alone

2. **Message signal model**
   - Purpose: judge the message around a link that already received verdict inform or warn
   - Input: surrounding message text
   - Output: scam-likelihood score + intent labels
   - Role: help determine whether surrounding context increases concern

## Design principle

Use small specialized models, not one general model.

Reason:
- URL classification is a different problem from message understanding
- smaller models are easier to run on device
- smaller models are easier to test, tune, and explain
- smaller models are safer as bounded signal generators

## Recommended first model split

### A. Link signal model

Use a tiny URL classifier.

Suggested starting direction:
- URL-specific model family
- tiny footprint
- optimized for phishing / malicious URL classification

What it should detect:
- suspicious token patterns
- deceptive brand-like text in path or subdomain
- unusual domain / path structure
- shortened or obfuscated forms
- suspicious TLD or lexical combinations

### B. Message signal model

Use a small multilingual text classifier.

Suggested starting direction:
- multilingual MiniLM or XLM-R style encoder
- fine-tuned for phishing / smishing / social engineering language

Why multilingual:
- LinkLook may need Dutch and English from the start
- language coverage matters more than raw benchmark prestige

### Recommended labels for the message model

Do not start with only safe / unsafe.
Use richer labels such as:
- benign
- urgent pressure
- credential request
- payment request
- impersonation
- account problem / technical support
- unclear but suspicious

This gives better product value and clearer user feedback.

## Decision flow

### 1. Primary verdict path
Primary verdict remains:
- policy
- rules
- heuristics
- backend reputation / known decisions

### 2. Link signal path
Run the URL model and produce:
- malicious probability
- confidence band
- top contributing pattern group

### 3. Message signal path
Only run when:
- verdict is inform or warn
- surrounding message text is available

Produce:
- scam-likelihood score
- message intent label(s)
- explanation tags for UX use

### 4. Final decision combiner
The combiner should be rule-based at first.

Example logic:
- if primary verdict is block → block
- if primary verdict is ok and AI is mildly suspicious → do not auto-escalate without strong evidence
- if primary verdict is inform and message model shows clear scam patterns → upgrade concern level in UX
- if primary verdict is warn and message model strongly confirms scam intent → strengthen warning language

## Important product rule

AI should not become the single source of truth.

Recommended order:
1. rules / policy / backend
2. URL model extra signal
3. message model extra signal
4. optional future LLM only for explanation

## Rollout order

### Phase 1
Add URL model only.
Goal:
- prove extra signal value on-device
- keep scope small
- measure false positives and false negatives

### Phase 2
Add message model only for inform and warn cases.
Goal:
- improve detection of scam context around ambiguous links
- avoid unnecessary cost and latency on all traffic

### Phase 3
Add explanation layer.
Goal:
- translate model signals into simple user-facing language
- keep the final verdict path deterministic

## Device strategy

**Broad support tier:** Base safety experience should work on all supported devices without depending on advanced AI hardware.

**Advanced AI tier:** Heavier or future AI features can be gated to newer devices.

## Inputs and outputs

### Link model input
- full URL
- normalized URL
- lexical features
- domain/path tokenization

### Link model output
- benign score
- suspicious score
- malicious score
- confidence band

### Message model input
- raw message text
- normalized text
- optional metadata such as channel type if available later

### Message model output
- benign / suspicious probability
- intent labels
- explanation tags

## Testing guidance

Test the models separately first.

### Link model tests
- obvious malicious URLs
- obvious benign URLs
- tricky lookalikes
- shortened URLs
- brand impersonation patterns

### Message model tests
- urgent payment requests
- fake bank / parcel / helpdesk messages
- normal benign sharing messages
- mixed Dutch / English examples
- short incomplete messages

### Combined tests
- benign link + benign message
- benign link + suspicious message
- suspicious link + benign message
- suspicious link + suspicious message

## What not to do yet
- do not start with one general chat model for both tasks
- do not allow AI alone to block or allow
- do not depend on Apple Intelligence-only features for the base product
- do not add explanation generation before the signal layers are stable

## Practical recommendation

Start with:
1. tiny URL classifier
2. multilingual message classifier
3. Core ML conversion and device testing
4. rule-based signal combiner

## One-sentence policy

LinkLook should use AI as a bounded, on-device extra signal layer: a tiny URL model for the link itself and a multilingual message model for surrounding text in inform and warn cases, while final verdicting remains controlled by deterministic rules and policy.
