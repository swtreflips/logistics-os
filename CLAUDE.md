# CLAUDE.md

Inbound logistics platform with an AI operations assistant built on top of it.

**Full documentation lives in [`OS/`](OS/README.md).** Read [`OS/ARCHITECTURE.md`](OS/ARCHITECTURE.md) before making architectural decisions, and [`OS/ROADMAP.md`](OS/ROADMAP.md) before assuming anything described is already built.

This file holds only the rules that apply while writing code.

---

# Repository Topology

This repo contains documentation only. The working code lives in sibling repos:

| Repo | Stack | Data |
|---|---|---|
| `rates-app` | Vite + React 18, JavaScript, MUI | Supabase, **direct from browser** |
| `stufferPlanner` | Vite + React 19, TypeScript, Radix + ag-Grid | none — CSV and client state |
| `logistics-api` | Fastify on Railway | not built yet |

All share one Supabase project and one auth model.

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

**3. No `supabase.from()` outside `src/services/`.** Not in components, not in hooks, not in route handlers. Services return DTOs, never raw Postgres rows, and their signatures never leak Supabase types.

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

Every business table carries `created_by` and `updated_by`.

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
- `uuid` vs `bigint` for IDs (D4)
- Postgres schemas vs table prefixes for module separation (D5)
- Grid library and component system (D6, D7)
- The definition of `customer` (D9)
- Soft vs hard delete (D10)

Ask rather than guessing. A wrong guess here propagates into every table and every service.
