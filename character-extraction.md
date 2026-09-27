# Character Extraction

The extraction layer turns free-form chat into structured, queryable traits. It runs **separately** from the reply generation and writes to Firestore on each user turn.

---

## Why a Separate Layer

If extraction lived inside the conversational prompt:
- Chat quality would degrade when the model focused on JSON output
- Extraction errors would surface as awkward replies
- Trait taxonomy changes would require retuning the chat prompt

Separating them makes each layer independently tunable and testable.

---

## Trait Taxonomy

### Static traits
Directly stated, low ambiguity:

| Trait | Type | Example |
|---|---|---|
| `age` | integer | 40 |
| `height_cm` | integer | 181 |
| `weight_kg` | integer | 90 |
| `gender` | enum | `"man"` / `"woman"` / `"other"` |
| `orientation` | enum | `"straight"` / `"gay"` / `"bi"` |
| `location_city` | string | `"Manchester"` |
| `intent` | enum | `"serious"` / `"casual"` / `"unsure"` |

### Soft traits
Inferred from language, tone, and content:

| Trait | Type | Example |
|---|---|---|
| `hobbies` | string[] | `["fishing", "travel", "art"]` |
| `values` | string[] | `["honesty", "independence"]` |
| `communication_style` | enum | `"direct"` / `"warm"` / `"playful"` |
| `openness` | 0.0–1.0 | `0.7` |
| `lifestyle_pace` | enum | `"active"` / `"calm"` |
| `relationship_history` | enum | `"experienced"` / `"new"` |

### Dealbreakers
Explicit non-negotiables. Extraction only writes these when the user states them clearly:

- "I'm not gay" → `orientation = "straight"` + `dealbreaker: same_gender`
- "No smokers" → `dealbreaker: smoking`
- "Must be over 25" → `dealbreaker: age_min: 25`
- "I want a woman" → `preference: gender = woman`

### Meta
- `confidence[trait]` — 0.0–1.0 per trait
- `last_updated` — timestamp per trait
- `source_message_id` — provenance for debugging
- `extraction_version` — prompt version that produced this trait

---

## Extraction Prompt Contract

Input:
```json
{
  "recent_user_messages": ["...", "..."],
  "existing_traits": { ... },
  "prompt_version": "v3"
}
Output (strict JSON):

json
{
  "traits": {
    "age": 40,
    "height_cm": 181,
    "hobbies": ["fishing", "travel"],
    "intent": "serious"
  },
  "dealbreakers": ["smoking"],
  "confidence": {
    "age": 0.95,
    "hobbies": 0.8,
    "intent": 0.6
  },
  "corrections": [
    { "trait": "gender_preference", "from": "man", "to": "woman" }
  ]
}
The extraction prompt enforces:

No speculation — if the user didn't say it, don't extract it

Corrections over overwrites — if a new value contradicts an old one, emit a correction record

Confidence as a first-class output — never emit a trait without a confidence score

Strict JSON — no prose, no markdown fences, no trailing commentary

Confidence Scoring
Confidence	Meaning
0.9–1.0	Explicitly stated, unambiguous
0.7–0.9	Clearly implied, little room for doubt
0.5–0.7	Inferred from context, may need confirmation
< 0.5	Weak signal — not written to Firestore
The match layer only reads traits with confidence ≥ 0.7. Lower-confidence traits are stored but flagged, so Aya can ask a confirming question later.

How confidence is assigned
Source	Base confidence
Direct statement ("I am 40")	0.95
Explicit correction by user	0.95
Clear implication ("I turned 40 last month")	0.85
Pattern across multiple turns	0.75
Single oblique mention	0.55
Inferred from tone/style	0.5
Confidence decays slowly over time if the trait hasn't been reconfirmed — confidence = base × decay(days_since_update), floor 0.4.

Corrections
When a new extraction contradicts a stored trait, the pipeline writes a corrections entry rather than silently overwriting. This gives:

An audit trail for debugging

A way to detect prompt drift

A signal to Aya that she should reflect the correction back to the user

Corrections also update the trait's confidence — a corrected trait starts at 0.95 (the user corrected it deliberately).

Correction flow
text
1. Extraction detects conflict: stored.gender_preference = "man", new = "woman"
2. Extraction emits correction record
3. Cloud Function updates users/{userId}.traits.gender_preference = "woman"
4. Cloud Function writes audit entry to users/{userId}/corrections/{id}
5. Extraction sets confidence["gender_preference"] = 0.95
6. Next chat turn: Aya reflects the correction back
   → "Got it — you're looking for a woman. Let me update that."
Failure Modes
Failure	Behavior
Gemini returns malformed JSON	Extraction skipped; chat reply unaffected
Trait contradicts two prior values	Correction recorded; new value stored with 0.6 confidence
User changes intent mid-conversation	Treated as a correction, not an error
Extraction latency > 5s	Function returns; traits written asynchronously on next turn
Model invents a trait not in message	Detected by source_message_id audit; trait dropped, alert logged
Prompt version mismatch	Trait written with extraction_version; re-extracted on next turn if schema changed
Extraction Versioning
Every extraction output is tagged with prompt_version. When the prompt changes:

New extractions use the new version

Existing traits keep their old extraction_version

A background job can re-extract old chats with the new prompt if needed

Diffs between versions surface in logs — this is how prompt drift is caught

Version bumps happen on:

Taxonomy changes (new trait, removed trait)

Confidence formula changes

Output schema changes

Any change that would produce different results on the same input

text

---

## 📄 `match-algorithm.md`

```markdown
# Match Algorithm

The match layer is deterministic. It runs in Cloud Functions, reads only high-confidence traits from Firestore, and never invokes the LLM.

---

## Inputs

For the querying user:
- Static traits (age, gender, orientation, location)
- Soft traits (values, hobbies, communication style)
- Dealbreakers
- Intent (serious / casual / unsure)

For each candidate:
- Same fields, from their Firestore record

---

## Hard Filters

Applied first. Any failure removes the candidate:

| Filter | Rule |
|---|---|
| Location | Must be in the same operating city |
| Intent | Must match (serious ↔ serious, casual ↔ casual) |
| Orientation | Mutual — A's preference includes B, and B's includes A |
| Dealbreakers | Neither user's dealbreakers fire on the other |
| Confidence gate | At least `intent`, `orientation`, `location` must be ≥ 0.7 |
| Recency | Candidate active within last 30 days |

Hard filters run in Firestore queries where possible, in-memory otherwise.

---

## Scoring Model

After hard filters, candidates are scored:
score = 0.30 × values_overlap

0.25 × lifestyle_similarity

0.20 × communication_style_match

0.15 × hobbies_overlap

0.10 × age_proximity

text

### Component definitions

- **values_overlap** — Jaccard similarity of `values[]`
- **lifestyle_similarity** — 1.0 if `lifestyle_pace` matches, 0.5 if adjacent, 0 otherwise
- **communication_style_match** — 1.0 exact match, 0.6 for "warm ↔ playful", 0 otherwise
- **hobbies_overlap** — Jaccard similarity, capped at 1.0
- **age_proximity** — `1 - |ageA - ageB| / 20`, floored at 0

Weights are tunable via Firestore config, so they can be adjusted without redeploying functions.

---

## Thresholds

| Score | Action |
|---|---|
| ≥ 0.75 | Top match — Aya proposes introduction |
| 0.60–0.75 | Candidate — held for future if top declines |
| 0.45–0.60 | Weak — not introduced, may be rescored as traits evolve |
| < 0.45 | Discarded |

Aya never introduces more than one candidate at a time to a given user. Silence is better than a weak match.

---

## Tie-Breaking

When scores are within 0.02:
1. Prefer the candidate with more complete traits
2. Then the one with a more recent `last_updated`
3. Then the one who has been waiting longer in the pool

Deterministic, so the same inputs always produce the same ranking.

---

## Feedback Loop

After a date, `postDateFeedback` collects:

- Did the date happen?
- How did it go? (1–5)
- Would you meet again? (yes / no / maybe)

Feedback adjusts:

- **Per-trait weights** for the user — traits that predicted a good date are up-weighted for that user's future matches
- **Candidate quality signal** — a candidate who consistently produces good dates rises in the pool
- **Dealbreaker detection** — if the user says "no" and cites a specific trait, that trait becomes a soft dealbreaker

Feedback never retroactively changes a match record — it only affects future scoring.

---

## What the LLM Does Here

Nothing. The LLM contributes in two places only:

1. **Reply generation** (conversation-design.md)
2. **Trait extraction** (character-extraction.md)

Everything downstream — filtering, scoring, ranking, tie-breaking, feedback weighting — is deterministic Node.js code. This is deliberate: matches must be explainable and reproducible.

---

## Explainability

For every match record, the function stores a `score_breakdown` object:

```json
{
  "values_overlap": 0.82,
  "lifestyle_similarity": 1.0,
  "communication_style_match": 0.6,
  "hobbies_overlap": 0.5,
  "age_proximity": 0.9,
  "total": 0.79
}
If a user asks Aya why she proposed a match, the conversational layer can read this and answer honestly.

text

---

## 📄 `engineering-notes.md`

```markdown
# Engineering Notes

Trade-offs and lessons behind Aya Matchmaker.

---

## Why Firebase

The whole backend runs on Firebase because:

- **Zero ops** — no servers, no containers, no scaling logic
- **Free tier covers production** — hosting, auth, functions, Firestore all fit in Spark/Blaze free quotas
- **Anonymous auth out of the box** — critical for "chat before signup"
- **Realtime listeners** — chat UI updates with no polling

The trade-off: Firebase locks the project into Google's ecosystem, and Firestore's query model is restrictive for complex match queries. We accepted the lock-in for development speed.

---

## Why Gemini, Not a Custom Model

- Free tier is generous enough for production conversation volume
- Long context handles full chat history without summarization tricks
- Handles both structured JSON output and open-ended chat with the same API

The trade-off: we don't control latency, and quota limits are opaque. If the free tier tightens, the cost curve is unpredictable.

---

## Why a Separate Extraction Pass

The single most important architectural decision.

Early prototype had the chat prompt return both a reply *and* a JSON trait blob. It failed because:

- The model degraded reply quality when juggling two outputs
- Extraction errors surfaced as weird chat messages
- Prompt changes for one goal broke the other

Splitting them doubled the function cost per turn but made each layer independently debuggable. Worth it.

---

## Trait Confidence as a First-Class Field

Storing `confidence[trait]` rather than a single number meant:

- The match layer can query `confidence >= 0.7` and ignore noise
- Aya can ask a confirming question when confidence is 0.5–0.7
- Debugging is easier — we can see *why* a match was or wasn't proposed

Cost: every extraction writes more fields, and the extraction prompt is longer. Also worth it.

---

## Lessons Learned

### 1. Match before the profile is "complete"
Waiting for the user to fill out every trait stalls onboarding. Matching on high-confidence traits early and refining over time works better.

### 2. Reflection beats interrogation
Simply restating what the user said before asking the next question roughly doubled the number of turns users were willing to complete.

### 3. Consent gates are non-negotiable
Early versions hinted at the other user's identity. Two users complained. Removed entirely.

### 4. Corrections are features, not bugs
When a user corrects a trait (e.g., orientation), the correction itself is high-confidence signal. Treating it that way improved match quality.

### 5. Prompt drift is real
The extraction prompt silently changed behavior across Gemini model updates. Storing `source_message_id` and diffing outputs across versions caught it.

---

## Known Constraints

| Constraint | Mitigation |
|---|---|
| No token TTL on anonymous sessions | Upgrade flow prompts registered auth after N turns |
| Gemini latency spikes | Chat reply is streamed; extraction is async |
| Firestore query limits | Candidate pool pre-filtered by city + intent before scoring |
| Single-city operation (Manchester) | Waitlist for other cities; Aya still offers advice |

---

## What I'd Do Differently

- **Start with the extraction layer first.** The conversational layer is easier to iterate on once traits are structured.
- **Version the extraction prompt from day one.** Retrofitting versioning was painful.
- **Write the match algorithm before any AI integration.** Deterministic logic is easier to reason about in isolation.
- **Test with real conversations earlier.** Real users ask things a synthetic test never would.

---

## Stack Summary

| Layer | Choice | Why |
|---|---|---|
| Hosting | Firebase Hosting | CDN, custom domain, free |
| Frontend | React + Firebase SDK | Fastest path to realtime chat |
| Backend | Cloud Functions (Node.js) | Serverless, callable, Firestore-native |
| Database | Firestore | Realtime, serverless, auth-integrated |
| Auth | Firebase Auth | Anonymous + upgrade flow |
| AI | Gemini API | Long context, structured output |
| Analytics | Firebase Analytics | Free, built-in |

---

**Author:** Ongun Akay
**Status:** ✅ Live in production