# CLAUDE.md

Inbound logistics platform with an AI operations assistant built on top of it.

**Full documentation lives in [`OS/`](OS/README.md).** Read [`OS/ARCHITECTURE.md`](OS/ARCHITECTURE.md) before making architectural decisions, and [`OS/ROADMAP.md`](OS/ROADMAP.md) before assuming anything described is already built.

This file holds only the rules that apply while writing code.

---

# Repository Topology

This repo contains documentation only. The working code lives in sibling repos:

| Repo | Stack | Where queries live |
|---|---|---|
| `Schedules/React` | Vite + React, TypeScript | `src/lib/supabase.ts` — named capability functions over RPCs |
| `rates-app` | Vite + React 18, JavaScript, MUI | `src/features/*/services/` — one service module per feature |
| `stufferPlanner` | Vite + React 19, TypeScript, Radix + ag-Grid | `src/data/repos/` — interfaces, with Local and Supabase implementations |
| `geoapi` | Next.js on Vercel | own authenticated routes — the GeoBrain service, **live** |
| `logistics-api` | Fastify on Railway | not built yet |

All share one Supabase project and one auth model. Inbound exists only as an early `mapLibre`
spike. The Python carrier scrapers feed Schedules and are deliberately kept separate.

**Three names for one concept.** `services/`, `lib/`, `repos/` are the same seam in three
dialects. Use one name in the next app. Converging the existing three is not worth a dedicated
effort, but each will need translating when `logistics-api` lands.

---

# Current Phase

The standalone apps are the foundation of the platform. The AI is a layer that will sit on top of them later.

Build for humans. Each app must be fully usable with no AI present, and no UI should contain chat boxes, prompt bars, or gaps reserved for a future assistant.

Build the *backend* as though the AI is certain — services, events, stable IDs, permissions in services. That groundwork is ordinary good practice, not a concession to a future feature.

There is no tradeoff between the two. If a decision seems to trade "good for users" against "ready for AI," one of them is being done wrong.

---

# Non-Negotiable Rules

**1. The database is the source of truth.** Never store shipment state in prompts or memory. Never let the AI infer it.

**2. Business logic exists exactly once, in a service.** Not in React, not in an API route, not in a prompt.

**3. No `supabase.from()` outside the data layer.** Not in components, not in hooks, not in route handlers. The layer returns DTOs, never raw Postgres rows, and its signatures never leak Supabase types.

**This rule is currently met.** All three apps keep queries behind `services/`, `lib/` or `repos/`; the only direct calls outside it are two identity lookups in `rates-app`'s `AuthProvider`, which is shell code. Keep it that way — the rule now exists to prevent regression, not to describe a cleanup.

**Rule 2 is the one that is not met.** Business rules — permission checks, filters, derivations — still live in React components across all three apps. Moving a query behind a function does not move the *rule* behind it. When you touch a component containing a decision about who may see or do what, that decision moves into the data layer with the query. Method in [`OS/ROADMAP.md`](OS/ROADMAP.md) Part 5.

**4. Every sentence the AI produces must trace to a field a tool returned.** If it cannot, say the platform does not record it.

**5. Anything with business consequences requires human approval.** The AI drafts; a person commits.

---

# Code

TypeScript everywhere. Strict. No `any`.

New files in `rates-app` are `.ts`/`.tsx` — do not convert existing JavaScript, just stop adding to it.

Small files, small services, composition over inheritance, consistent naming.

Independent lookups inside a service run in `Promise.all`, not sequential `await`s.

## Database

`snake_case`, plural tables, `<entity>_id` foreign keys, `timestamptz` in UTC.

Module-private tables carry a module prefix — `planner_`, `drayage_`, `sched_`. Genuinely shared
entities do not: `organizations`, `profiles`, `notifications`. Older `rates` and `schedules`
tables predate the convention and are not worth renaming.

Every **new** business table carries `created_by` and `updated_by`. Today **no table has them** —
30 tables, zero. That is the single largest unmet item in these documents, and the only one that
gets worse by waiting, because the history it would have recorded is lost as it accrues.

Never overwrite shipment status — append an event. Current state is derived from events and materialized into a projection; nothing writes the projection directly.

## Errors

Never expose stack traces. Every service defines its not-found and permission-denied shapes explicitly. A permission denial must not reveal whether the record exists.

---

# Stack

Vite SPAs on Vercel, one subdomain per module. Fastify API on Railway. Supabase for data and auth. Anthropic Claude with LangGraph for AI.

Next.js is used for the GeoBrain service only — never for frontends or the business API.

Maps are reached only through GeoBrain, never by calling HERE or OpenStreetMap directly.

---

# Do Not Assume

These are undecided. Check `OS/ROADMAP.md` Part 3 before writing code that depends on them.

- Whether the API forwards the user's JWT or uses the service role key (D2)
- What identity background jobs and AI agents run as (D3)
- Grid library and component system (D6, D7)
- The definition of `customer` (D9) — see [`OS/GLOSSARY.md`](OS/GLOSSARY.md)
- Soft vs hard delete (D10)

Settled by practice, recorded in `OS/ROADMAP.md` Part 3: **D4** IDs are `uuid` with
`gen_random_uuid()`; **D5** module separation is table prefixes, not Postgres schemas.

Ask rather than guessing. A wrong guess here propagates into every table and every service.
