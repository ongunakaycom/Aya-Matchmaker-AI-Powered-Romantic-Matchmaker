# character-extraction.md

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
```

Output (strict JSON):
```json
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
```

The extraction prompt enforces:
- **No speculation** — if the user didn't say it, don't extract it
- **Corrections over overwrites** — if a new value contradicts an old one, emit a correction record
- **Confidence as a first-class output** — never emit a trait without a confidence score
- **Strict JSON** — no prose, no markdown fences, no trailing commentary

---

## Confidence Scoring

| Confidence | Meaning |
|---|---|
| 0.9–1.0 | Explicitly stated, unambiguous |
| 0.7–0.9 | Clearly implied, little room for doubt |
| 0.5–0.7 | Inferred from context, may need confirmation |
| < 0.5 | Weak signal — not written to Firestore |

The match layer only reads traits with **confidence ≥ 0.7**. Lower-confidence traits are stored but flagged, so Aya can ask a confirming question later.

### How confidence is assigned

| Source | Base confidence |
|---|---|
| Direct statement ("I am 40") | 0.95 |
| Explicit correction by user | 0.95 |
| Clear implication ("I turned 40 last month") | 0.85 |
| Pattern across multiple turns | 0.75 |
| Single oblique mention | 0.55 |
| Inferred from tone/style | 0.5 |

Confidence decays slowly over time if the trait hasn't been reconfirmed — `confidence = base × decay(days_since_update)`, floor 0.4.

---

## Corrections

When a new extraction contradicts a stored trait, the pipeline writes a `corrections` entry rather than silently overwriting. This gives:

- An audit trail for debugging
- A way to detect prompt drift
- A signal to Aya that she should reflect the correction back to the user

Corrections also update the trait's confidence — a corrected trait starts at 0.95 (the user corrected it deliberately).

### Correction flow

```
1. Extraction detects conflict: stored.gender_preference = "man", new = "woman"
2. Extraction emits correction record
3. Cloud Function updates users/{userId}.traits.gender_preference = "woman"
4. Cloud Function writes audit entry to users/{userId}/corrections/{id}
5. Extraction sets confidence["gender_preference"] = 0.95
6. Next chat turn: Aya reflects the correction back
   → "Got it — you're looking for a woman. Let me update that."
```

---

## Failure Modes

| Failure | Behavior |
|---|---|
| Gemini returns malformed JSON | Extraction skipped; chat reply unaffected |
| Trait contradicts two prior values | Correction recorded; new value stored with 0.6 confidence |
| User changes intent mid-conversation | Treated as a correction, not an error |
| Extraction latency > 5s | Function returns; traits written asynchronously on next turn |
| Model invents a trait not in message | Detected by `source_message_id` audit; trait dropped, alert logged |
| Prompt version mismatch | Trait written with `extraction_version`; re-extracted on next turn if schema changed |

---

## Extraction Versioning

Every extraction output is tagged with `prompt_version`. When the prompt changes:

1. New extractions use the new version
2. Existing traits keep their old `extraction_version`
3. A background job can re-extract old chats with the new prompt if needed
4. Diffs between versions surface in logs — this is how prompt drift is caught

Version bumps happen on:
- Taxonomy changes (new trait, removed trait)
- Confidence formula changes
- Output schema changes
- Any change that would produce different results on the same input