# Logistics OS

An inbound logistics platform with an AI operations assistant built on top of it.

> **Build a logistics platform, not an AI application.**
> The AI is simply another consumer of the platform.

---

# The Foundation

**The standalone apps are the platform. The AI is a layer that uses them.**

Every module — Schedules, Rates, Stuffer Planner, Inbound — must be fully valuable to a human operator with no AI involved at all. If the AI layer were never built, the platform would still be worth having.

That is the standard, and it is the current phase: the apps are in active development, human-only, and correctly so.

## No AI-Shaped Holes

While building for humans, do **not**:

- Add chat boxes, prompt bars, or "ask AI" affordances
- Design schemas around what a language model might find convenient
- Build features that only make sense once an assistant exists
- Leave gaps in the UI intended to be filled by AI later

A dashboard that needs an assistant to be usable is a broken dashboard.

## But Lay The Groundwork

The AI layer needs nothing exotic. It needs the things a well-built platform has anyway:

| Groundwork | Why humans need it | Why AI needs it |
|---|---|---|
| Business logic in services, not components | Testable, reusable across pages | Tools call services — they cannot call a React component |
| Events instead of overwritten status | Real history; "what changed since yesterday?" | Explaining *why* something happened |
| Normalized data with stable IDs | Correct joins, no duplicated text | Reliable entity resolution |
| Permissions enforced in services | Correct access control | The AI inherits the user's access, never exceeds it |
| Named capabilities (`getContainerTimeline`) | Clear, reusable code | Becomes a tool unchanged |
| Recorded reasons, not just outcomes | Operators can explain a delay to a customer | The assistant can too — and stays silent when there is no reason on file |

## The Point

**There is no tradeoff.**

The discipline that makes an app good for humans is the same discipline that makes it ready for AI. Clear capabilities, a real data model, no business logic in components — these are not concessions to a future assistant. They are what a well-built application looks like.

If a decision seems to trade "good for users" against "ready for AI," one of the two is being done wrong.

Build the human app correctly and the groundwork is already laid.

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
| **[SECURITY.md](SECURITY.md)** | Threat model, standards, what to get right during the build, and what to wire up per layer |
| **[DESIGN.md](DESIGN.md)** | What every module holds constant, and the one thing each varies |
| **[GLOSSARY.md](GLOSSARY.md)** | One meaning per domain term, across every module and service |
| **[ROADMAP.md](ROADMAP.md)** | What is decided, what is not, what is currently wrong, and what to do next |

Read ARCHITECTURE first. Everything else assumes it.

---

# Current Reality

The platform does not exist yet as a single system. It is being built as standalone apps that share one Supabase project and one auth model, surfaced to internal users through a hub of cards.

| | Schedules | rates-app | stufferPlanner | geoapi | logistics-api |
|---|---|---|---|---|---|
| Stack | Vite + React, TS | Vite + React 18, JS | Vite + React 19, TS | Next.js | Fastify |
| Data access | `lib/supabase.ts` | `features/*/services/` | `data/repos/` | own routes | not built |
| Deploy | Vercel | Vercel | Vercel | Vercel — **live** | Railway (planned) |

**This is a deliberate sequence, not an accident.** Shared database and shared auth are the hard parts of consolidation, and they already exist.

**The data layer also exists** — in all three frontends, under three different names. Every business query is behind a named function or a repository interface; none are issued from a React component. That is the substance of Rule 3, and it was reached by ordinary good practice rather than by a migration project.

Two things are genuinely missing, and they are different from each other:

1. **The extracted API.** The data layer runs *in the browser*, so a cron job, a webhook, or an AI tool has nothing to call. This is the P2 gate.
2. **Business rules still live in components.** Permission checks, filters, and derivations sit in React alongside the code that renders them. That is Rule 2, and it is the gap that actually causes bugs today — see `ROADMAP.md` Part 2.

`ROADMAP.md` tracks the gap between this reality and the architecture described here. Read it before assuming any document describes something that has been built.

---

# A Note On Tense

These documents describe the target architecture in the present tense. Most of it is not built.

Where a document says "the API does X," read it as "the API will do X, and nothing else may do X in the meantime."
