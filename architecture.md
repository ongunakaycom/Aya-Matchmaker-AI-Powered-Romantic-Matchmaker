# architecture.md

# System Architecture

Aya Matchmaker is a serverless, event-driven system built entirely on Firebase. Every user interaction is a Cloud Function call; every piece of state lives in Firestore.

---

## High-Level Flow

```
┌──────────────────────┐
│   React SPA          │
│   Firebase Hosting   │
└──────────┬───────────┘
           │ HTTPS (Firebase SDK)
           ▼
┌──────────────────────┐
│  Cloud Functions     │
│  Node.js runtime     │
└──────────┬───────────┘
           │
     ┌─────┼─────────────┬───────────────┐
     ▼     ▼             ▼               ▼
┌─────────┐ ┌────────┐ ┌─────────┐ ┌──────────┐
│Firestore│ │Gemini  │ │Firebase │ │Analytics │
│         │ │API     │ │Auth     │ │          │
└─────────┘ └────────┘ └─────────┘ └──────────┘
```

---

## Components

### 1. Frontend (React SPA)
- Loaded from Firebase Hosting (CDN-backed)
- Auth via Firebase SDK (anonymous first, upgrade later)
- All backend interaction through callable Cloud Functions
- No direct Firestore writes for chat or match state

### 2. Cloud Functions
Stateless Node.js functions. Each function is a discrete unit:

| Function | Trigger | Purpose |
|---|---|---|
| `chat` | HTTPS callable | User message → Aya reply |
| `extractProfile` | Firestore write on `chats/{id}/messages` | Chat turn → structured traits |
| `findMatch` | HTTPS callable | Traits → ranked candidates |
| `contactMatch` | HTTPS callable | Send intro to a candidate |
| `scheduleDate` | HTTPS callable | Coordinate time/place |
| `postDateFeedback` | HTTPS callable | Collect feedback, update weights |

### 3. Firestore
Source of truth for all persistent state.

### 4. Gemini API
Used for two things only:
- Generating Aya's conversational reply
- Extracting structured traits from chat text

Never used for matchmaking decisions.

### 5. Firebase Auth
Anonymous sessions by default. Users can upgrade to email/Google without losing chat history.

---

## Firestore Schema

```
users/{userId}
  ├── createdAt: timestamp
  ├── authType: "anonymous" | "email" | "google"
  ├── displayName: string
  ├── location: { city, region, country }
  ├── intent: "serious" | "casual" | "unsure"
  └── traits: {
        static: { age, height, gender, ... }
        soft: { hobbies[], values[], style }
        dealbreakers: [string]
        confidence: { [trait]: 0.0–1.0 }
      }

chats/{chatId}
  ├── userId: string
  ├── startedAt: timestamp
  └── lastMessageAt: timestamp

chats/{chatId}/messages/{messageId}
  ├── role: "user" | "aya"
  ├── text: string
  ├── createdAt: timestamp
  └── extracted: { traits snapshot, confidence delta }

matches/{matchId}
  ├── userA: string
  ├── userB: string
  ├── score: number
  ├── status: "pending" | "contacted" | "accepted" | "declined" | "dated"
  ├── createdAt: timestamp
  └── feedback: { userA, userB } (post-date)

dates/{dateId}
  ├── matchId: string
  ├── scheduledAt: timestamp
  ├── location: string
  └── status: "proposed" | "confirmed" | "completed" | "cancelled"
```

---

## Request Lifecycle — Chat Turn

```
1. User types message in SPA
2. SPA → chat() callable with { chatId, text }
3. chat() writes user message to Firestore
4. chat() reads recent history + user traits
5. chat() calls Gemini with conversational prompt
6. chat() writes Aya reply to Firestore
7. Firestore trigger fires extractProfile() on new user message
8. extractProfile() calls Gemini with extraction prompt
9. extractProfile() merges traits + confidence into users/{userId}
10. SPA listener renders Aya reply
```

Chat and extraction are **decoupled**. A slow or failing extraction never blocks the user's reply.

---

## Request Lifecycle — Match

```
1. findMatch() reads user's traits above confidence threshold
2. Queries candidate users by location + intent
3. Scores each candidate against traits
4. Returns top N ranked candidates (never to the client directly — only to the match layer)
5. contactMatch() sends an intro message to the top candidate
6. Candidate's Aya asks them if they're interested
7. If both consent → scheduleDate() runs
```

---

## Failure Modes

| Failure | Behavior |
|---|---|
| Gemini timeout on `chat` | Fallback message; user message still persisted |
| Gemini timeout on `extractProfile` | Silently skipped; retried on next turn |
| Firestore unavailable | Function returns error; SPA shows retry prompt |
| Match candidate declined | Match record updated; user notified neutrally |

---

## Cost Profile

| Component | Free tier | Notes |
|---|---|---|
| Firebase Hosting | 10 GB / month | Sufficient for MVP |
| Cloud Functions | 2M invocations / month | Chat + extraction = 2 per turn |
| Firestore | 50k reads / 20k writes per day | Extraction triggers dominate reads |
| Gemini API | Free tier quotas | Latency-bound, not cost-bound |

# conversation-design.md

# Conversational Design

Aya's conversational layer has one job: **turn a stranger into a well-modeled user** without feeling like an interrogation.

---

## Design Principles

### 1. One question per turn
Every Aya reply contains at most one explicit question. Two questions in one message causes users to answer only the easier one, wasting a turn.

### 2. Reflect before you ask
Before moving to a new topic, Aya restates what she heard:
> "So you're into fishing and travel — got it. What does a perfect weekend look like for you?"

Reflection doubles as confirmation and makes the user feel heard.

### 3. No fabrication
Aya never invents:
- Names of other users
- Statistics about compatibility
- Details the user hasn't shared

If she doesn't know something, she says so and asks.

### 4. Consent gates
The other user's identity is never revealed until both sides have agreed to be introduced. Aya declines politely and offers to wait.

### 5. Tone adaptation
Aya reads the user's tone and matches it:

| User tone | Aya tone |
|---|---|
| Playful / emoji-heavy | Warm, light, playful |
| Formal / terse | Direct, no filler |
| Vulnerable | Gentle, slower, fewer questions |
| Testing / trolling | Patient, no engagement with provocation |

---

## Elicitation Ladder

Aya doesn't ask for traits in a fixed order. She picks the **highest-signal unanswered question** given the user's state.

Priority order:

1. **Intent** — serious, casual, or unsure?
2. **Orientation & preference** — who are they looking for?
3. **Location** — must be in an operating city
4. **Non-negotiables** — dealbreakers stated clearly
5. **Values** — what matters to them in a partner
6. **Lifestyle** — work, hobbies, schedule
7. **Soft preferences** — age range, height range, personality type

Aya stops climbing the ladder once trait confidence crosses the matching threshold. She does not "complete" a profile for its own sake.

---

## Prompt Strategy

Three separate prompts, each with a single responsibility:

| Prompt | Input | Output |
|---|---|---|
| **Reply** | Recent history + user traits | Aya's next message |
| **Extract** | Last N user messages | Structured JSON traits |
| **Reflect** | Newly extracted traits | Optional confirmation line |

Keeping prompts separate means:
- The conversational prompt stays short and cheap
- The extraction prompt can be tuned independently
- A bad extraction never degrades the chat reply

---

## Boundaries

Aya will not:

- Reveal another user's identity, contact info, or location
- Promise a match within a timeframe
- Give medical, legal, or financial advice
- Continue a conversation that becomes abusive (she ends it and logs the turn)
- Match a user outside their stated orientation or dealbreakers

Aya will:

- Ask for clarification when a message is ambiguous
- Acknowledge her own mistakes (e.g., wrong pronoun)
- Offer relationship advice when asked
- Say "I don't know yet" when she genuinely doesn't

---

## Anti-Patterns

Things the prompt explicitly avoids:

- **Multi-question walls** — "What's your age? Height? Location? Hobbies?"
- **Form language** — "Please provide your date of birth"
- **Fake empathy** — "I understand how you feel" without any basis
- **Premature matching** — offering a match before intent and location are known
- **Overpromising** — "I'll find you someone by Friday"

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
| `gender` | enum | `"man"` |
| `orientation` | enum | `"straight"` |
| `location_city` | string | `"Manchester"` |
| `intent` | enum | `"serious"` |

### Soft traits
Inferred from language, tone, and content:

| Trait | Type | Example |
|---|---|---|
| `hobbies` | string[] | `["fishing", "travel", "art"]` |
| `values` | string[] | `["honesty", "independence"]` |
| `communication_style` | enum | `"direct"` / `"warm"` / `"playful"` |
| `openness` | 0.0–1.0 | `0.7` |
| `lifestyle_pace` | enum | `"active"` / `"calm"` |

### Dealbreakers
Explicit non-negotiables. Extraction only writes these when the user states them clearly:

- "I'm not gay" → `orientation = "straight"` + `dealbreaker: same_gender`
- "No smokers" → `dealbreaker: smoking`
- "Must be over 25" → `dealbreaker: age_min: 25`

### Meta
- `confidence[trait]` — 0.0–1.0 per trait
- `last_updated` — timestamp per trait
- `source_message_id` — provenance for debugging

---

## Extraction Prompt Contract

Input:
```json
{
  "recent_user_messages": ["...", "..."],
  "existing_traits": { ... }
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

---

## Confidence Scoring

| Confidence | Meaning |
|---|---|
| 0.9–1.0 | Explicitly stated, unambiguous |
| 0.7–0.9 | Clearly implied, little room for doubt |
| 0.5–0.7 | Inferred from context, may need confirmation |
| < 0.5 | Weak signal — not written to Firestore |

The match layer only reads traits with **confidence ≥ 0.7**. Lower-confidence traits are stored but flagged, so Aya can ask a confirming question later.

---

## Corrections

When a new extraction contradicts a stored trait, the pipeline writes a `corrections` entry rather than silently overwriting. This gives:

- An audit trail for debugging
- A way to detect prompt drift
- A signal to Aya that she should reflect the correction back to the user

Corrections also update the trait's confidence — a corrected trait starts at 0.95 (the user corrected it deliberately).

---

## Failure Modes

| Failure | Behavior |
|---|---|
| Gemini returns malformed JSON | Extraction skipped; chat reply unaffected |
| Trait contradicts two prior values | Correction recorded; new value stored with 0.6 confidence |
| User changes intent mid-conversation | Treated as a correction, not an error |
| Extraction latency > 5s | Function returns; traits written asynchronously on next turn |

# match-algorithm.md

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

```
score = 0.30 × values_overlap
      + 0.25 × lifestyle_similarity
      + 0.20 × communication_style_match
      + 0.15 × hobbies_overlap
      + 0.10 × age_proximity
```

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

Everything downstream — filtering, scoring, ranking, tie-breaking, feedback weighting — is deterministic Python/Node code. This is deliberate: matches must be explainable and reproducible.

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
```

If a user asks Aya *why* she proposed a match, the conversational layer can read this and answer honestly.

# engineering-notes.md

# Engineering Notes

Trade-offs, lessons, and the roadmap behind Aya Matchmaker.

---

## Why Firebase

The whole backend runs on Firebase because:

- **Zero ops** — no servers, no containers, no scaling logic
- **Free tier covers MVP** — hosting, auth, functions, Firestore all fit in Spark/Blaze free quotas
- **Anonymous auth out of the box** — critical for "chat before signup"
- **Realtime listeners** — chat UI updates with no polling

The trade-off: Firebase locks the project into Google's ecosystem, and Firestore's query model is restrictive for complex match queries. We accepted the lock-in for MVP speed.

---

## Why Gemini, Not a Custom Model

- Free tier is generous enough for MVP conversation volume
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
| No automated tests | Manual test script per function before each deploy |

---

## Roadmap

### Near term
- Automated tests for extraction + match scoring
- Multi-city support with per-city candidate pools
- Post-date feedback UI

### Mid term
- Voice input for chat
- "Why this match?" — Aya explains score breakdown on request
- Cross-session memory so Aya remembers long-term context

### Long term
- Group introductions (friends-of-friends)
- Community events — Aya coordinates small group meetups
- Partnership with local venues for date suggestions

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