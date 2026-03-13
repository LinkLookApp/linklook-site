# AI Model Input/Output Contracts

> Exact specifications for LinkLook's two AI models.
> Code and tests can be written against these contracts without ambiguity.
> See also: AI Architecture Note v1, CLAUDE.md rules 41–42.

---

## Model A: Link Signal Model (URL Classifier)

### Purpose
Extra signal for the link itself. Runs on every URL check for Pro users.
Detects patterns that rule-based checks may miss: subtle combinations of
features that individually look benign but together signal phishing.

### Deployment
- **Runtime:** Core ML on-device (Apple Neural Engine / CPU fallback)
- **Trigger:** Every URL check (Pro users only)
- **Timeout:** Must complete within the 8-second checking window
- **Privacy:** No data leaves the device

### Input: `URLModelInput`

The model receives a **fixed-length feature vector** derived from the
`NormalizedURL` that already exists in the codebase.

```
Field                     Type      Description
─────────────────────────────────────────────────────────────────
fullURL                   String    Original URL string (for tokenization)
hostTokens                [String]  Host split on "." and "-"
pathTokens                [String]  Path split on "/" and "-"
registrableDomain         String?   From NormalizedURL
subdomain                 String?   From NormalizedURL
tld                       String?   From NormalizedURL
subdomainDepth            Int       Number of subdomain levels (0–N)
pathDepth                 Int       Number of path segments
urlLength                 Int       Total character count of original URL
hostLength                Int       Character count of host
domainEntropy             Float     Shannon entropy of registrable domain
pathEntropy               Float     Shannon entropy of path
hasIPAddress              Bool      From NormalizedURL.isIPAddress
isPunycode                Bool      From NormalizedURL.isPunycode
hasUserInfo               Bool      From NormalizedURL.hasUserInfo
hasNonStandardPort        Bool      port != nil && port != 80 && port != 443
isHTTPS                   Bool      scheme == "https"
queryParamCount           Int       Number of query parameters
percentEncodedCount       Int       Number of %-encoded sequences in URL
containsBrandToken        Bool      Any known brand appears in host tokens
brandTokenPosition        String    "none" | "start" | "middle" | "subdomain"
suspiciousKeywordCount    Int       Count of phishing keywords in domain+path
tldCategory               String    "common" | "country" | "new" | "suspicious"
```

**Derived features** (computed by the feature extractor, not the model):
- `domainEntropy`: Shannon entropy calculated over characters in registrable domain
- `pathEntropy`: Shannon entropy calculated over characters in path
- `tldCategory`: Mapped from existing TLD lists in the codebase
- `brandTokenPosition`: Mapped from existing brand detection logic

### Output: `URLModelOutput`

```
Field                     Type      Range       Description
──────────────────────────────────────────────────────────────────
benignScore               Float     0.0 – 1.0   P(benign)
suspiciousScore           Float     0.0 – 1.0   P(suspicious but not malicious)
maliciousScore            Float     0.0 – 1.0   P(malicious/phishing)
confidence                Float     0.0 – 1.0   Model self-assessed confidence
topPatternGroup           String                 Most influential feature group
```

Scores sum to 1.0 (softmax output).

`topPatternGroup` values (one of):
- `"domain_structure"` — subdomain depth, entropy, length
- `"brand_mimicry"` — brand tokens in unusual positions
- `"keyword_pattern"` — phishing keywords in domain/path
- `"obfuscation"` — encoding, punycode, IP address, userinfo
- `"tld_risk"` — suspicious or unusual TLD
- `"path_structure"` — deep paths, suspicious path tokens
- `"mixed_signals"` — no single dominant group

### Decision thresholds

```
maliciousScore >= 0.85  AND  confidence >= 0.7   →  signal: aiUrlRiskHigh
maliciousScore >= 0.60  AND  confidence >= 0.8   →  signal: aiUrlRiskMedium (future)
otherwise                                        →  signal: aiUrlClean (internal only)
```

`aiUrlRiskHigh` can escalate: OK → INFORM, INFORM → WARN. Never overrides BLOCK.
`aiUrlClean` is internal — never shown to user, never downgrades.

### Training data
- Public phishing datasets: PhishTank, OpenPhish
- Legitimate URL corpus: Tranco top-10k, Alexa top-10k
- Must be validated against LinkLook's existing golden test fixtures before release

---

## Model B: Message Signal Model (Context Classifier)

### Purpose
Judge the surrounding message text when a link received verdict INFORM or WARN.
Helps detect social engineering patterns that URL analysis alone cannot see.
This is the "Check Message Context" Pro feature (CLAUDE.md rules 20–22).

### Deployment
- **Runtime:** Core ML on-device (multilingual NLP encoder)
- **Trigger:** Only when user taps "Check Message Context" on INFORM/WARN
- **Privacy:** Text processed on-device, never stored after analysis (CLAUDE.md rule 21)
- **Languages:** Dutch and English from the start; architecture supports adding more

### Input: `MessageModelInput`

```
Field                     Type      Description
──────────────────────────────────────────────────────────────────
rawText                   String    The message text the user pasted
normalizedText            String    Lowercased, whitespace-normalized
tokenCount                Int       Word count of normalized text
channelHint               String?   "sms" | "email" | "chat" | "unknown"
                                    (optional — from InputContext.sourceType)
```

The model performs its own tokenization internally (multilingual encoder).
`channelHint` is metadata only — helps the model calibrate expectations
(SMS scams tend to be shorter and more urgent than email scams).

### Output: `MessageModelOutput`

```
Field                     Type      Range       Description
──────────────────────────────────────────────────────────────────
benignScore               Float     0.0 – 1.0   P(legitimate message)
scamLikelihood            Float     0.0 – 1.0   P(scam/social engineering)
confidence                Float     0.0 – 1.0   Model self-assessed confidence

intentLabels              [ScoredLabel]          Top intent labels with scores
explanationTags           [String]               Human-readable tags for UX
```

`benignScore + scamLikelihood` sum to 1.0.

### Intent labels (CLAUDE.md rule 20)

Each label has a score (0.0–1.0) and a one-line explanation.

```
Label                     Description
──────────────────────────────────────────────────────────────────
urgency_pressure          "Creates urgency or time pressure"
credential_request        "Asks for login details or personal info"
payment_request           "Asks for money or payment action"
impersonation             "Pretends to be a known organization"
account_problem           "Claims there's a problem with your account"
emotional_manipulation    "Uses fear, excitement, or sympathy"
grammar_anomalies         "Contains unusual grammar or formatting"
benign_sharing            "Normal message sharing a link"
unclear_suspicious        "Cannot classify clearly, but patterns are unusual"
```

### ScoredLabel type

```swift
struct ScoredLabel: Sendable {
    let label: String       // e.g. "urgency_pressure"
    let score: Float        // 0.0 – 1.0
    let explanation: String // e.g. "Creates urgency or time pressure"
}
```

### Decision thresholds (UX mapping)

The message model produces a traffic-light verdict for UX (CLAUDE.md rule 22):

```
scamLikelihood >= 0.75  AND  confidence >= 0.7
    →  "This message shows strong signs of a scam"         (red)

scamLikelihood >= 0.40  AND  confidence >= 0.6
    →  "Some warning signs detected"                       (orange)

otherwise
    →  "This message looks normal"                         (green)
```

Always append caveat: "This analysis is based on the text you provided.
It doesn't guarantee the link is safe." (CLAUDE.md rule 22)

### UX display
- Show traffic-light verdict (one sentence)
- Show top 3–4 scored intent labels with explanations
- Labels are sorted by score descending, only show score > 0.15
- Context check adds information — never overrides the URL verdict

---

## Signal Combiner (Rule-Based)

The combiner is **deterministic**, not ML. It merges AI signals into
the existing verdict pipeline.

### URL model combiner rules

```
Rule    Condition                                    Action
────────────────────────────────────────────────────────────────────
U1      aiUrlRiskHigh + primary verdict OK       →   Escalate to INFORM
        Reason: "LinkLook's AI model detected suspicious patterns"

U2      aiUrlRiskHigh + primary verdict INFORM   →   Escalate to WARN
        Reason: same as U1

U3      aiUrlRiskHigh + primary verdict WARN     →   Keep WARN (already escalated)
        Add signal badge: "AI-detected patterns"

U4      aiUrlRiskHigh + primary verdict BLOCK    →   Keep BLOCK (never override)
        Add signal only

U5      aiUrlClean + any primary verdict         →   No change (never downgrade)
        Internal signal only

U6      AI timeout or unavailable                →   No change
        Use structural + GSB results only
```

### Message model combiner rules

The message model does NOT change the URL verdict. It produces a separate
context analysis result displayed alongside the verdict.

```
Rule    Condition                                    Action
────────────────────────────────────────────────────────────────────
M1      scamLikelihood >= 0.75                   →   Show red verdict + labels
M2      scamLikelihood >= 0.40                   →   Show orange verdict + labels
M3      scamLikelihood < 0.40                    →   Show green verdict + labels
M4      Model timeout or error                   →   Show "Could not analyze"
```

The context check result is displayed on a separate screen — it does not
modify buttons, verdict color, or primary recommendation on the verdict screen.

---

## Swift Protocol Contracts

These are the protocol signatures that the Core ML wrappers must implement.

### URL model protocol

```swift
public protocol URLSignalModelProtocol: Sendable {
    func predict(input: URLModelInput) async -> URLModelOutput
}

public struct URLModelInput: Sendable {
    public let fullURL: String
    public let normalizedURL: NormalizedURL
    // Feature extractor derives all other fields from NormalizedURL
}

public struct URLModelOutput: Sendable {
    public let benignScore: Float
    public let suspiciousScore: Float
    public let maliciousScore: Float
    public let confidence: Float
    public let topPatternGroup: String
}
```

### Message model protocol

```swift
public protocol MessageSignalModelProtocol: Sendable {
    func predict(input: MessageModelInput) async -> MessageModelOutput
}

public struct MessageModelInput: Sendable {
    public let rawText: String
    public let channelHint: String?
}

public struct MessageModelOutput: Sendable {
    public let benignScore: Float
    public let scamLikelihood: Float
    public let confidence: Float
    public let intentLabels: [ScoredLabel]
    public let explanationTags: [String]
}

public struct ScoredLabel: Sendable {
    public let label: String
    public let score: Float
    public let explanation: String
}
```

---

## Testing contract

Both models must pass the following test categories before release:

### URL model test matrix
| Category              | Expected output          | Min accuracy |
|-----------------------|--------------------------|-------------|
| Known phishing URLs   | maliciousScore > 0.8     | 90%         |
| Trusted brand URLs    | benignScore > 0.8        | 95%         |
| Lookalike domains     | maliciousScore > 0.6     | 80%         |
| Unknown neutral URLs  | No false block            | 99%         |
| LinkLook golden fixtures | Match existing verdicts | 100%        |

### Message model test matrix
| Category                    | Expected output             | Min accuracy |
|-----------------------------|-----------------------------|-------------|
| Urgent payment scams        | scamLikelihood > 0.7        | 85%         |
| Bank/parcel impersonation   | impersonation label > 0.6   | 85%         |
| Normal link sharing         | benignScore > 0.7           | 90%         |
| Dutch scam messages         | Correct language handling    | 80%         |
| Short/incomplete messages   | confidence < 0.5 (low)      | 80%         |
