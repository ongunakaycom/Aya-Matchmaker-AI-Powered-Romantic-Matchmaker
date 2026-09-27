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

Aya's job is not to answer questions about dating — it's to build a working model of the user. From a live session:

> **User:** "40 years old man 181cm 90k"
> **Aya:** "Okay, great! So you're a 40-year-old man, 181cm tall, and 90kg. That's a good start! 😊 To help me find your perfect match, could you tell me a little more about what you're looking for in a woman?"

The bot:
- Extracts structured attributes from free-form messages
- Reflects them back to confirm
- Asks the next highest-signal question
- Never invents a match before enough signal is collected
- Refuses to expose the other user's identity until both sides consent

---

## 🔌 Cloud Functions Surface

Firebase Functions handle the full backend:

| Function | Responsibility |
|---|---|
| `chat` | Receives a user message, calls Gemini, returns Aya's reply |
| `extractProfile` | Parses chat turns into structured traits in Firestore |
| `findMatch` | Scores candidates against extracted traits |
| `contactMatch` | Sends the introduction to a potential match |
| `scheduleDate` | Coordinates time/place once both sides agree |

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

---

## 📚 Documentation

| File | Contents |
|---|---|
| `architecture.md` | System design, Firestore schema, function flow |
| `conversation-design.md` | Prompt strategy, personality extraction rules |
| `engineering-notes.md` | Trade-offs, lessons learned, roadmap |

---

## 📜 License

MIT — see [LICENSE](./LICENSE).