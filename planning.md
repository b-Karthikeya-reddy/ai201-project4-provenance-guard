# Provenance Guard — Planning Document

## Architecture

### Submission Flow

```
POST /submit
    │
    ▼
[Input Validation]
(text, creator_id required)
    │
    ▼
[Signal 1: LLM Classification]
Groq llama-3.3-70b-versatile
Returns score 0.0–1.0 (1.0 = AI)
    │
    ▼
[Signal 2: Stylometric Heuristics]
Sentence length variance + type-token ratio + punctuation density
Returns score 0.0–1.0 (1.0 = AI)
    │
    ▼
[Confidence Scoring]
Weighted average: 0.6 * LLM + 0.4 * stylometric
Maps to: likely_ai / uncertain / likely_human
    │
    ▼
[Transparency Label Generator]
Produces human-readable label text based on confidence
    │
    ▼
[Audit Log]
Writes structured JSON entry (timestamp, content_id, scores, label, status)
    │
    ▼
JSON Response → client
(content_id, attribution, confidence, label)
```

### Appeal Flow

```
POST /appeal
    │
    ▼
[Lookup content_id in log]
    │
    ▼
[Update status → "under_review"]
    │
    ▼
[Append appeal_reasoning to audit log entry]
    │
    ▼
JSON Response → client
(appeal_received: true, status: "under_review")
```

### Architecture Narrative

A submitted piece of text passes through two independent detection signals — an LLM semantic classifier and a stylometric heuristic analyzer — whose outputs are combined into a single weighted confidence score. That score drives both the transparency label shown to the user and the structured audit log entry written for every decision. If a creator contests the result, the appeal endpoint updates the content's status and appends the creator's reasoning to the existing log entry for human review.

---

## Detection Signals

### Signal 1: LLM Classification (Groq)

**What it measures:** Semantic and stylistic coherence holistically — whether the text "reads" like AI output. Captures things like unnaturally smooth transitions, generic hedging language ("it is important to note that"), overuse of parallel structure, and lack of personal specificity.

**Output:** A score between 0.0 and 1.0, where 1.0 = high confidence AI-generated. Extracted from a structured JSON response from the model.

**Blind spot:** A skilled human writer who drafts formal, polished prose (academic writing, legal documents) may score high. Also, AI text that has been lightly edited by a human will confuse this signal. The LLM is judging style, not provenance.

**Prompt approach:**
```
You are an AI content detection assistant. Analyze the following text and return 
a JSON object with one field: "ai_score" (float 0.0–1.0, where 1.0 means 
highly confident this is AI-generated and 0.0 means highly confident this is 
human-written). Consider: uniformity of sentence structure, presence of generic 
filler phrases, lack of personal voice, and overly balanced hedging.

Text: {text}

Return ONLY valid JSON: {"ai_score": 0.XX}
```

---

### Signal 2: Stylometric Heuristics

**What it measures:** Statistical surface properties that differ between human and AI writing. AI writing tends to be more uniform; human writing is more variable. Specifically:

- **Sentence length variance:** AI text has low variance (sentences tend to be similar lengths). Human text is more irregular.
- **Type-token ratio (TTR):** Vocabulary diversity. AI text reuses common words more; human writing varies more. TTR = unique words / total words.
- **Punctuation density:** Human writers use dashes, ellipses, informal punctuation. AI text uses commas and periods more uniformly.

**Output:** A score between 0.0 and 1.0 (1.0 = AI-like uniformity). Computed in pure Python, no external libraries.

**Blind spot:** Short texts (<50 words) produce unreliable stylometric scores — variance calculations need enough data points to be meaningful. Also, highly formal human writing (academic papers, legal briefs) naturally has low variance and will score high.

---

## Confidence Scoring and Uncertainty Representation

**Combination formula:**
```
confidence = (0.6 * llm_score) + (0.4 * stylometric_score)
```

LLM signal gets higher weight because it captures semantic properties that stylometrics miss. Stylometrics serve as a structural check, especially for shorter or borderline texts.

**What scores mean:**

| Score Range | Meaning | Label Category |
|-------------|---------|----------------|
| 0.75 – 1.00 | High confidence AI | `likely_ai` |
| 0.40 – 0.74 | Uncertain — signals disagree or text is ambiguous | `uncertain` |
| 0.00 – 0.39 | High confidence human | `likely_human` |

**Why this threshold:** The uncertain band is deliberately wide (0.40–0.74) because false positives — labeling a human's work as AI — are worse than false negatives on a creative platform. We'd rather surface "uncertain" than wrongly accuse a creator.

A score of 0.51 and a score of 0.95 produce meaningfully different labels: the first triggers the uncertain label with an appeal prompt, the second triggers the high-confidence AI label. They are never treated the same.

---

## Transparency Label Variants

All three variants are shown verbatim below — these are the exact strings returned in API responses and displayed to users.

**High-confidence AI (confidence ≥ 0.75):**
```
⚠️ AI-Generated Content Detected
Our system is highly confident this content was generated by an AI tool 
rather than written by a human. Confidence: {confidence}%.
If you are the creator and believe this is incorrect, you may file an appeal.
```

**Uncertain (confidence 0.40–0.74):**
```
🔍 Authorship Uncertain
Our system could not confidently determine whether this content was 
human-written or AI-generated. Confidence: {confidence}%.
This content has been flagged for transparency. The creator may file an 
appeal to provide additional context.
```

**High-confidence human (confidence < 0.40):**
```
✅ Likely Human-Written
Our system believes this content was written by a human creator. 
Confidence: {100 - confidence}%.
Attribution signals indicate original human authorship.
```

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted content (identified by creator_id).

**What they provide:**
- `content_id` — the ID returned at submission
- `creator_reasoning` — a free-text explanation (e.g., "I am a non-native English speaker and my formal writing style may resemble AI output")

**What the system does:**
1. Looks up the content_id in the audit log
2. Updates `status` from `"classified"` to `"under_review"`
3. Appends `appeal_reasoning` and `appeal_timestamp` to the log entry
4. Returns confirmation JSON to the creator

**What a human reviewer sees:**
- Original content text
- Both signal scores and combined confidence
- The transparency label that was shown
- The creator's appeal reasoning
- Timestamp of original classification and appeal

**No automated re-classification.** Appeals go to a human review queue. The system does not automatically change the attribution result.

---

## Anticipated Edge Cases

**Edge case 1: Short texts (<50 words)**
A haiku or a two-sentence caption doesn't give stylometric heuristics enough data to compute meaningful variance. Sentence length variance with 2 sentences is meaningless. The system will lean entirely on the LLM signal for short texts, which increases uncertainty. Mitigation: flag texts under 50 words and widen the uncertain band for them.

**Edge case 2: Formal human writing**
An academic abstract or legal brief written by a human will have low sentence length variance, high structural uniformity, and formal vocabulary — all traits the stylometric signal associates with AI. A human PhD student submitting their own abstract could easily score in the uncertain or likely_ai range. This is a known false positive risk and is why the appeal workflow exists.

**Edge case 3: Lightly edited AI output**
If a user generates text with an AI tool and then edits it — fixing typos, adding a personal anecdote, breaking up sentences — the text may score mid-range on both signals. The system will correctly surface "uncertain" rather than confidently mislabeling it. This is acceptable behavior.

---

## API Surface

| Endpoint | Method | Input | Output |
|----------|--------|-------|--------|
| `/submit` | POST | `{text, creator_id}` | `{content_id, attribution, confidence, label}` |
| `/appeal` | POST | `{content_id, creator_reasoning}` | `{appeal_received, status}` |
| `/log` | GET | — | `{entries: [...]}` |

Rate limiting applied to `/submit`: **10 requests per minute, 100 per day per IP.**

Reasoning: A legitimate creator submits their own work — rarely more than a few times per hour. 10/minute allows normal usage while blocking automated flooding. 100/day is generous for a single creator but stops large-scale scraping.

---

## AI Tool Plan

### Milestone 3 (Submission endpoint + Signal 1)
- **Spec sections provided:** Detection signals (Signal 1), Architecture diagram, API surface table
- **What I'll ask for:** Flask app skeleton with POST /submit route stub + Groq LLM signal function that returns a 0–1 score
- **Verification:** Call the signal function directly with the 4 test inputs from the spec. Check that clearly AI text scores >0.7 and clearly human text scores <0.4 before wiring into the endpoint.

### Milestone 4 (Signal 2 + Confidence scoring)
- **Spec sections provided:** Detection signals (Signal 2), Uncertainty representation, Architecture diagram
- **What I'll ask for:** Stylometric heuristics function + weighted confidence scoring logic matching the 0.6/0.4 weights and threshold ranges above
- **Verification:** Run all 4 test inputs through both signals separately. Print both scores to confirm they're producing different values. Check that combined score maps to correct label category per the threshold table.

### Milestone 5 (Production layer)
- **Spec sections provided:** Transparency label variants (exact text), Appeals workflow, Architecture diagram, Rate limiting section
- **What I'll ask for:** Label generation function mapping score ranges to exact label text + POST /appeal endpoint
- **Verification:** Submit inputs targeting all three confidence ranges and confirm all three label variants are returned. Submit an appeal and call GET /log to confirm status updated to "under_review" and reasoning is captured.
