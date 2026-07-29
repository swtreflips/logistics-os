# Logistics OS

An inbound logistics platform with an AI operations assistant built on top of it.

> **Build a logistics platform, not an AI application.**
> The AI is simply another consumer of the platform.

---

# The Five Rules

Everything else in these documents follows from these. If a decision seems ambiguous, resolve it against this list.

**1. The database is the source of truth. The AI is not.**

Shipment state lives in PostgreSQL. It is never stored in prompts, never inferred, never remembered between turns.

**2. Business logic exists exactly once, in a service.**

Not in React. Not in an API route. Not in a prompt. Not in a report. One implementation, many consumers.

**3. The frontend never queries the database.**

Neither does the AI. Both call services. Services own the rules; repositories own the data.

**4. Every sentence the AI produces must trace to a field a tool returned.**

If it cannot, the assistant says the platform does not record it. A fluent answer that outruns its data is worse than no answer — it is wrong in a way the user cannot detect.

**5. Anything with business consequences requires human approval.**

The AI may draft, summarize, and recommend. It may not send, modify, delete, or release.

---

# The Guiding Question

Every time a new feature is proposed:

> **If we removed the React UI tomorrow, would this capability still exist?**

If no, it was built in the wrong layer.

---

# Documents

| Document | Owns |
|---|---|
| **[ARCHITECTURE.md](ARCHITECTURE.md)** | Layering, service boundaries, the two deployables, technology stack, hosting, data model philosophy, security, engineering standards |
| **[MODULES.md](MODULES.md)** | The four business domains, platform services, and the lifecycle that connects them |
| **[AI.md](AI.md)** | AI philosophy, tool design, grounding rules, phase roadmap, and a worked example |
| **[ROADMAP.md](ROADMAP.md)** | What is decided, what is not, what is currently wrong, and what to do next |

Read ARCHITECTURE first. Everything else assumes it.

---

# Current Reality

The platform does not exist yet as a single system. It is being built as standalone apps that share one Supabase project and one auth model, surfaced to internal users through a hub of cards.

| | rates-app | stufferPlanner | logistics-api |
|---|---|---|---|
| Stack | Vite + React 18, JavaScript | Vite + React 19, TypeScript | not built |
| Data | Supabase, direct from browser | none — CSV and client state | — |
| Deploy | Vercel | Vercel | Railway (planned) |

**This is a deliberate sequence, not an accident.** Shared database and shared auth are the hard parts of consolidation, and they already exist. What is missing is the service layer.

`ROADMAP.md` tracks the gap between this reality and the architecture described here. Read it before assuming any document describes something that has been built.

---

# A Note On Tense

These documents describe the target architecture in the present tense. Most of it is not built.

Where a document says "the API does X," read it as "the API will do X, and nothing else may do X in the meantime."
