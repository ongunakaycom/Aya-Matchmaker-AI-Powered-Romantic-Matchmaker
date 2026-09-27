 engineering-notes.md

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