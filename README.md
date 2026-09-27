# Aya Matchmaker — AI-Powered Romantic Matchmaker

Source code is private. This repository documents the architecture, conversational design, and engineering decisions behind a production-deployed AI matchmaker that replaces swiping and forms with real conversation.

**Live:** https://aya-matchmaker.web.app

---

## 🎯 What it does

Aya is an AI matchmaker that gets to know users through natural conversation — no swiping, no profile forms. She learns your personality, interests, and what you're looking for, then introduces compatible people and helps set up the actual date.

Unlike dating apps that match on self-reported profile data, Aya extracts character signals from conversation and uses them for compatibility scoring.

**Design principle:** The LLM drives conversation and personality analysis, but match logic, user state, and message routing stay in deterministic cloud functions. The model never owns state.

---

## 📐 System at a Glance

```
React SPA (Firebase Hosting)
        ↓
Cloud Functions (Node.js)
        ↓
   ├── Firestore (users, chats, matches)
   ├── Gemini API (character analysis + conversation)
   └── Firebase Auth (anonymous + registered)
```

---

## 🧩 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React, Firebase SDK |
| Backend | Firebase Cloud Functions (Node.js) |
| Database | Cloud Firestore |
| AI | Google Gemini API |
| Auth | Firebase Authentication |
| Hosting | Firebase Hosting |
| Analytics | Firebase Analytics |

---

## 🧠 Engineering Highlights

- **Conversation-first onboarding** — no signup wall before the first message; anonymous auth, upgrade later
- **Character extraction from dialogue** — Gemini analyzes chat turns and writes structured traits to Firestore
- **Deterministic match layer** — matchmaking runs in Cloud Functions on top of stored traits, not inside the LLM
- **Stateless conversation loop** — each turn is a function call; Firestore is the source of truth
- **Match handoff** — Aya contacts the other user, mediates interest, and coordinates the date
- **Zero-cost entry** — free tier covers chat, Firebase Spark plan covers hosting + auth

---

## 💬 Conversational Design

Aya is designed as a **character-elicitation agent**, not a Q&A bot. Every reply serves one of three goals:

| Goal | Behavior |
|---|---|
| **Elicit** | Ask the next highest-signal question (interests, values, dealbreakers) |
| **Reflect** | Confirm a trait before storing it |
| **Defer** | Politely avoid revealing match identity or other users' data |

The prompt layer enforces one question per turn, no fabrication, consent boundaries, and tone adaptation. Full strategy in [`conversation-design.md`](./conversation-design.md).

---

## 🧬 Character Extraction

The extraction pipeline runs on each turn and writes to Firestore:

- **Static traits** — age, height, gender, location, relationship intent
- **Soft traits** — hobbies, values, communication style, openness
- **Dealbreakers** — explicit non-negotiables stated by the user
- **Confidence** — per-trait confidence score, updated as the user reveals more

The match layer only consumes traits above a minimum confidence threshold, which prevents premature matching on weak signal. Full taxonomy in [`character-extraction.md`](./character-extraction.md).

---

## 🔌 Cloud Functions Surface

| Function | Responsibility |
|---|---|
| `chat` | Receives a user message, calls Gemini, returns Aya's reply |
| `extractProfile` | Parses chat turns into structured traits in Firestore |
| `findMatch` | Scores candidates against extracted traits |
| `contactMatch` | Sends the introduction to a potential match |
| `scheduleDate` | Coordinates time/place once both sides agree |
| `postDateFeedback` | Collects feedback after a date and adjusts future matches |

Full flow diagrams in [`architecture.md`](./architecture.md).

---

## 📊 System Maturity

| Layer | Status |
|---|---|
| Chat · Auth · Hosting · Cloud Functions | ✅ Production |
| Profile extraction · Match scoring | 🟡 Tuning |
| Date scheduling · Post-date feedback | 🔴 Roadmap |
| Automated testing | 🔴 Roadmap |

---

## 🚧 Known Constraints

- Match quality depends heavily on how much the user shares in chat
- Gender/preference parsing occasionally misfires (documented in logs)
- Aya only operates in Manchester; other cities are waitlist-only
- No automated test coverage yet
- Session-level memory only; no long-term cross-session user model yet

---

## 📚 Documentation

| File | Contents |
|---|---|
| [`architecture.md`](./architecture.md) | System design, Firestore schema, Cloud Functions flow, failure modes, cost profile |
| [`conversation-design.md`](./conversation-design.md) | Prompt strategy, elicitation ladder, tone guide, boundaries, anti-patterns |
| [`character-extraction.md`](./character-extraction.md) | Trait taxonomy, extraction prompt contract, confidence scoring, corrections, versioning |

---

## 📜 License

MIT — see [LICENSE](./LICENSE).

**Author:** Ongun Akay
**Status:** ✅ Live in production