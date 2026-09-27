# System Architecture

Aya Matchmaker is a serverless, event-driven system built entirely on Firebase. Every user interaction is a Cloud Function call; every piece of state lives in Firestore.

---

## High-Level Flow
┌──────────────────────┐
│ React SPA │
│ Firebase Hosting │
└──────────┬───────────┘
│ HTTPS (Firebase SDK)
▼
┌──────────────────────┐
│ Cloud Functions │
│ Node.js runtime │
└──────────┬───────────┘
│
┌─────┼─────────────┬───────────────┐
▼ ▼ ▼ ▼
┌─────────┐ ┌────────┐ ┌─────────┐ ┌──────────┐
│Firestore│ │Gemini │ │Firebase │ │Analytics │
│ │ │API │ │Auth │ │ │
└─────────┘ └────────┘ └─────────┘ └──────────┘

text

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

text

---

## Request Lifecycle — Chat Turn
User types message in SPA

SPA → chat() callable with { chatId, text }

chat() writes user message to Firestore

chat() reads recent history + user traits

chat() calls Gemini with conversational prompt

chat() writes Aya reply to Firestore

Firestore trigger fires extractProfile() on new user message

extractProfile() calls Gemini with extraction prompt

extractProfile() merges traits + confidence into users/{userId}

SPA listener renders Aya reply

text

Chat and extraction are **decoupled**. A slow or failing extraction never blocks the user's reply.

---

## Request Lifecycle — Match
findMatch() reads user's traits above confidence threshold

Queries candidate users by location + intent

Scores each candidate against traits

Returns top N ranked candidates (never to the client directly — only to the match layer)

contactMatch() sends an intro message to the top candidate

Candidate's Aya asks them if they're interested

If both consent → scheduleDate() runs

text

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
| Firebase Hosting | 10 GB / month | Sufficient for production |
| Cloud Functions | 2M invocations / month | Chat + extraction = 2 per turn |
| Firestore | 50k reads / 20k writes per day | Extraction triggers dominate reads |
| Gemini API | Free tier quotas | Latency-bound, not cost-bound |