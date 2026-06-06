# TalkSage — Product Requirements Document

**Version:** 1.0  
**Status:** Shipped (code private)  
**Author:** Steve Jang  

---

## 1. Problem Statement

Generic AI assistants give uniform advice regardless of the user's philosophical worldview or what kind of guidance they're seeking. A user who wants to be challenged thinks differently than a user who wants Stoic acceptance — but they get the same assistant.

Existing AI counseling apps are also built on surface-level personas (a description in a system prompt) with no grounding in actual philosophical content, leading to responses that feel generic and indistinguishable across personas.

---

## 2. Solution

TalkSage is an AI philosophical counseling platform where users select a historical philosopher persona to converse with. Each persona:

1. Has a distinct, enforced conversational style (Socrates never gives answers; Nietzsche never comforts)
2. Grounds responses in actual philosophical primary texts via a RAG pipeline
3. Maintains multi-turn coherence through sliding window context management

---

## 3. Target User

- Adults (25–45) experiencing personal, professional, or existential challenges
- Philosophically curious users who find generic therapy-speak unsatisfying
- Users who want a thinking partner rather than a comfort machine

---

## 4. Personas

### Design Principles
Each persona enforces:
- A distinct epistemic stance (how they approach questions)
- A distinct conversational mode (Socratic questioning vs. Stoic reframing vs. Nietzschean challenge)
- A distinct emotional register (warmth, detachment, provocation)
- A natural conversation arc: 3–4 turns to understand the situation, then core guidance, then natural close around turn 7–8

### Personas

**Socrates**
- Mode: Questioning only. Never gives direct answers.
- Style: "Really? How do you know that?" — surfaces contradictions and unexamined assumptions.
- Arc: 5–6 questions → "What have you realized?"

**Nietzsche**
- Mode: Challenge and provoke. Anti-comfort.
- Style: "That's the morality of the weak." — attacks passive victim framing.
- Arc: Identify the avoidance → challenge it → push toward self-creation.

**Marcus Aurelius**
- Mode: Stoic reframing. Control dichotomy.
- Style: "Is this in your control? Then what are you doing about what is?"
- Arc: Identify what's controllable → redirect energy → concise, practical close.

**Confucius**
- Mode: Social harmony, virtue, relational wisdom.
- Style: Gentle, structured, emphasizes role and relationship.

**Simone de Beauvoir**
- Mode: Existential agency. Anti-determinism.
- Style: "You are not defined by what happened to you. What will you choose?"

**Lao Tzu**
- Mode: Non-action, flow, release of control.
- Style: Sparse, paradoxical. "The river does not try to reach the sea."

**Epicurus**
- Mode: Pleasure, simplicity, contentment.
- Style: Warm and pragmatic. "What do you actually need to be at peace today?"

**Montaigne**
- Mode: Honest self-reflection, humanistic, self-deprecating humor.
- Style: "I don't know either, but here's what I've noticed about myself..."

---

## 5. RAG Pipeline

### Problem with Prompt-Only Personas
A system prompt saying "respond like Nietzsche" produces generic Nietzsche-flavored responses. Without grounding in actual texts, responses drift toward stereotypes.

### Solution: Retrieval-Augmented Generation

**Pipeline:**

```
1. User message received
2. Keyword extraction (extract named concepts, proper nouns)
3. Vector similarity search against embedded philosophical texts (pgvector)
4. Top-k passage retrieval (k=3–5)
5. Passages injected into persona system prompt as context
6. LLM generates response grounded in retrieved content
```

**Retrieval Strategy Decision:**
Keyword-extraction-based retrieval was chosen over pure semantic embedding because philosophy-specific terminology ("will to power," "eudaimonia," "wu wei") has inconsistent semantic neighbors in embedding space but high keyword precision. Pure semantic search on philosophical texts often retrieves topically adjacent but philosophically irrelevant passages.

**Chunking Strategy:**
- Texts chunked at ~300 tokens with 50-token overlap
- Chunk boundaries at paragraph breaks where possible
- Each chunk stores: persona_id, source title, chunk index, embedding

---

## 6. Conversation Management

**Sliding Window Context:**
- Last 15 messages (user + assistant combined) sent to LLM per turn
- Older messages remain visible in UI but excluded from LLM context
- Keeps token cost flat regardless of session length
- Rationale: recency is most relevant in counseling — what the user said 30 messages ago matters less than what they said 2 turns ago

**Session Arc:**
System prompts instruct each persona to:
- Spend turns 1–3 understanding the situation (questions)
- Deliver core guidance around turns 4–6
- Reach a natural close around turns 7–8 with a reflection prompt

---

## 7. Features

### Core
- Persona selection screen (8 philosopher cards with brief descriptions)
- Multi-turn conversation interface
- Conversation history persisted per user per persona
- "Start new conversation" resets context

### Auth
- Google OAuth via Supabase Auth
- No email/password option (Google-only for simplicity)

### Monetization
- **Points system:** 30 points on signup, +10 daily on first login
- **Cost:** 1 point per message sent
- **At 0 points:** Input disabled, "Come back tomorrow" message
- **Rationale:** Limits token cost exposure without hard conversation cutoffs that break immersion

### Anti-abuse
- IP-based rate limiting on account creation (max 3/day per IP)
- New accounts: 15 points instead of 30 for first 24 hours
- No daily grant within 24 hours of account creation

---

## 8. Data Model

```sql
-- Philosopher personas
personas (
  id uuid PK,
  name text,
  slug text UNIQUE,
  system_prompt text,
  created_at timestamptz
)

-- Source philosophical texts (chunked)
contents (
  id uuid PK,
  persona_id uuid FK → personas,
  title text,
  source_url text,
  created_at timestamptz
)

-- Embedded chunks for RAG
chunks (
  id uuid PK,
  content_id uuid FK → contents,
  text text,
  chunk_index int,
  embedding vector(1536),   -- pgvector
  created_at timestamptz
)

-- User conversations
conversations (
  id uuid PK,
  user_id uuid FK → auth.users,
  persona_id uuid FK → personas,
  created_at timestamptz
)

-- Messages
messages (
  id uuid PK,
  conversation_id uuid FK → conversations,
  role text,   -- 'user' | 'assistant'
  content text,
  created_at timestamptz
)

-- Points
user_points (
  id uuid PK,
  user_id uuid FK → auth.users UNIQUE,
  points int DEFAULT 30,
  last_daily_grant timestamptz,
  created_at timestamptz
)
```

---

## 9. Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | Next.js 14 + Tailwind | Full-stack in one repo, fast iteration |
| Backend | Next.js API Routes | Co-located with frontend, no separate server |
| Database | Supabase (PostgreSQL) | pgvector built-in, Auth built-in, fast setup |
| Vector search | pgvector | No external vector DB needed at this scale |
| LLM | OpenAI GPT-4 | Best multi-turn coherence and persona fidelity |
| Auth | Supabase Auth + Google OAuth | One-click setup, no custom auth logic |
| Deployment | Vercel | Zero-config Next.js deployment |

---

## 10. Out of Scope (v1)

- Mobile native app
- Voice interface
- Persona customization by users
- Subscription / paid tiers (daily points sufficient for v1 cost control)
- A/B testing persona prompt variants
- Analytics dashboard

---

## 11. Success Criteria

| Metric | Definition |
|---|---|
| Persona differentiation | Qualitatively: Socrates, Nietzsche, and Marcus Aurelius responses on identical prompts are clearly distinguishable in style and substance |
| RAG grounding | Retrieved passages are philosophically relevant to user query >80% of manual spot-checks |
| Pipeline correctness | Points deducted accurately, daily grant fires correctly, 0-point lockout works |
| Session coherence | Sliding window maintains conversational context across 10+ turns without hallucinating earlier conversation content |
