# Roadmap

What is decided, what is not, what is currently wrong, and what to do next.

`ARCHITECTURE.md` describes the target. This document describes the distance to it.

---

# Part 1 — Stages and Phases

Three separate documents previously numbered their phases 1–5 on different axes, so "Phase 3" meant different things depending on which you read. Two distinct tracks:

## Platform Stages

| Stage | Meaning | Status |
|---|---|---|
| **P1** | Human dashboards. Fast, accurate, easy to use. No AI assumptions in the UI. | **in progress** |
| **P2** | Service layer extracted. Every dashboard operation is a reusable service behind an API. | not started |
| **P3** | Platform complete. All modules exposing capabilities; background automation running. | not started |

## P1 Is Where The Work Is

The standalone apps are the foundation of the platform, not a prototype of it. They are built for humans, and they must be complete for humans — if the AI layer were never built, the platform would still be worth having.

So P1 is not a phase to rush through on the way to the interesting part. It *is* the platform.

What P1 does carry is an obligation: build in the way that leaves P2 available. Every item in Part 4 below exists for that reason, and none of them require thinking about AI while designing a screen. They are ordinary good practice that happens to also be the entire prerequisite for an assistant.

**Nothing in P1 should contain an AI feature.** No chat boxes, no prompt bars, no UI gaps reserved for a future assistant. See `README.md` — *No AI-Shaped Holes*.

## AI Phases

| Phase | Meaning | Requires |
|---|---|---|
| **A1** | Read-only Inbound assistant | P2 |
| **A2** | Communication — drafts, human-approved | A1 |
| **A3** | Operational — tasks, monitoring, reports | A2 |
| **A4** | Planning intelligence across Schedules, Rates, Planner | A3 + P3 |
| **A5** | Cross-module reasoning and proactive monitoring | A4 |

**P2 is the gate.** No AI phase can begin before the service layer exists. Writing tools first means writing them twice.

Detail on each AI phase is in `AI.md`.

---

# Part 2 — Current State

The platform is being built as standalone apps sharing one Supabase project. This is a deliberate sequence, not an accident: shared database and shared auth are the hard parts of consolidation, and they already exist.

| | rates-app | stufferPlanner | logistics-api |
|---|---|---|---|
| Stack | Vite + React 18 | Vite + React 19 | not built |
| Language | JavaScript | TypeScript | — |
| UI | MUI + MUI X DataGrid | Radix + ag-Grid | — |
| Styling | Tailwind 3 | Tailwind 4 | — |
| State | — | zustand | — |
| Data | Supabase, **direct from browser** | none — CSV + client state | — |
| Deploy | Vercel | Vercel | Railway (planned) |

**Shared:** one Supabase project, one auth model, RLS working for internal vs. external users, hub of cards for internal navigation.

## The Real Problem

Not that the apps are separate. That **rates-app talks to Supabase directly from the browser**, so no service layer exists anywhere.

- It contradicts the layering in `ARCHITECTURE.md`
- It hard-blocks every AI phase — LangGraph cannot call a React component
- Background jobs have nowhere to live

This would be a problem in a single monolithic app too. Merging three SPAs that all query Supabase from the browser produces one bigger SPA that queries Supabase from the browser.

---

# Part 3 — Decisions Log

| # | Decision | Status |
|---|---|---|
| D1 | API layer | **closed** — standalone Fastify service on Railway |
| D2 | Authorization: RLS-authoritative or service-authoritative | open — urgent |
| D3 | Service identity for jobs and AI agents | open — urgent |
| D4 | ID strategy: uuid or bigint | open |
| D5 | Module separation: Postgres schemas or table prefixes | open |
| D6 | Standard grid library | open |
| D7 | Standard component system: MUI or Radix | open |
| D8 | Domain strategy | **closed** — subdomains per module, `api.domain.com` |
| D9 | Definition of `customer` | open |
| D10 | Soft delete or hard delete | open |
| D11 | GeoBrain consumers: API only, frontends too, or both | open |
| D12 | GeoBrain cache store: shared Supabase schema or dedicated | open |

## D2 — Authorization

Currently RLS does everything, because the browser talks to Postgres directly. Once Fastify sits in between:

- **User JWT forwarded** — RLS stays authoritative, API adds business rules. Least disruption, but jobs and agents have no JWT.
- **Service role key** — API becomes authoritative, RLS bypassed server-side. More power, but every rule RLS gives you free must be reimplemented and tested.

Most systems land on: user JWT for interactive requests, service role for jobs and agents, services enforcing rules in both paths.

**Decide before the first Fastify route.**

## D11 and D12 — GeoBrain

Neither blocks anything today; both get expensive once GeoBrain exists.

**D11** — two consumers means two cache-invalidation paths and two rate-limit enforcement points. **API-only is the reversible choice**; opening it to frontends later is easy, closing it afterward is not.

**D12** — a geocoding cache will accumulate millions of rows. In shared Supabase it needs its own schema (see D5) and must be excluded from the audit and RLS conventions that exist for business data. A dedicated store sidesteps that.

## D9 — `customer`

For an inbound importer this is genuinely ambiguous: end retailer, DC, or delivery recipient? Half the planned AI queries depend on it. Define it before three apps define it differently.

---

# Part 4 — Groundwork

The cost of a gap is not its size. It is how much it **compounds**.

| Compounds badly | Doesn't compound |
|---|---|
| Schema and naming decisions | Which grid library |
| Direct DB access spreading through components | Which CSS framework |
| Identity and tenancy model | Monorepo vs. polyrepo |
| Domain vocabulary drift | React version |
| Missing audit columns | Folder layout |

The right column looks like the mess. The left column determines whether consolidation takes two weeks or two quarters.

## Tier 1 — Do Now

### 1.1 Stop writing `supabase.from()` inside React components

**The most important item here.**

You don't need the backend yet. You need the **call sites shaped** so a backend can slide underneath.

```js
// src/services/rateService.js
// Still calls Supabase from the browser today.
// The signature is what matters — it does not leak Supabase types.
export async function getRatesForLane(laneId) {
  const { data, error } = await supabase
    .from('rates').select('*').eq('lane_id', laneId);
  if (error) throw new ServiceError('RATES_LOOKUP_FAILED', error);
  return data.map(toRateDTO);
}
```

The component calls `getRatesForLane(laneId)` and knows nothing else. When the API lands, the body becomes a `fetch` and **no component changes**. Same when a LangGraph tool needs it — it imports the service, not the query.

Rules:
- No `supabase.from()` outside `src/services/`
- Services return DTOs, never raw Postgres rows
- Signatures use domain types, never Supabase client types
- Existing violations: leave them, convert opportunistically

### 1.2 Generate and share database types

```bash
supabase gen types typescript --project-id <id> > packages/types/src/db.ts
```

Highest leverage per hour on this list. Publish as `@logistics/types` via a `file:` dependency — no monorepo tooling required.

### 1.3 Freeze schema conventions

ID strategy · `created_at`/`updated_at` as `timestamptz` UTC · `created_by`/`updated_by` on every business table · soft-delete policy · `snake_case`, plural tables, `<entity>_id` foreign keys · module ownership via schema or prefix.

Retrofitting audit columns loses all prior history. Adding them now is free.

### 1.4 Answer D2 and D3

See Part 3.

### 1.5 Write a glossary

If rates says `lane`, stuffer says `route`, and inbound says `shipment_leg` for the same thing, consolidation becomes a translation project.

`lane` · `route` · `leg` · `shipment` · `booking` · `container` · `customer` · `supplier` · `forwarder` · `carrier` · `port` · `origin`/`destination` · `ETA`/`ETD`/`ATA`/`ATD` · `cargo ready` · `item`/`SKU`

Costs an hour. Cheapest item on this list.

### 1.6 Stop adding JavaScript to rates-app

Add `tsconfig.json` with `allowJs: true`. Write new files as `.ts`. Convert nothing.

## Tier 2 — Soon

**2.1 Stand up `logistics-api`.** The change that unlocks AI, jobs, and cross-module queries. If 1.1 is done consistently, it's mechanical.

Consequences to plan for: cross-origin calls need CORS and a token strategy (D2); two deploy targets means API changes must stay backward-compatible; the API needs its own repo, env management, and server-side Supabase client.

**2.2 Give stufferPlanner persistence.** It cannot participate in anything until it has a schema. Not a consolidation cost — unfinished work that exists regardless.

**2.3 Shared UI package.** Once two apps need the same component. Not before.

**2.4 Event log for Inbound.** Build it right the first time: events table, projection maintained on insert, no mutable status column. See `ARCHITECTURE.md` Part 5.

## Tier 3 — Explicitly Deferred

Not doing these is correct. Listed so they don't read as oversights.

- **Monorepo** — shared packages via `file:` deps get 90% of the benefit at 10% of the cost
- **Frontend stack convergence** — freeze new divergence, converge opportunistically; two grid libraries is ugly, not dangerous
- **Next.js for frontends or the business API** — ruled out (D1). Used for GeoBrain only; that exception must not spread.
- **Sentry / OpenTelemetry** — add when there's production traffic worth observing
- **Anything LangGraph** — blocked on 2.1 regardless

---

# Part 5 — Next Steps

Ordered. Early steps are decisions and documents that cost no code and stop further divergence. Code work is last and deliberately small.

**None of this requires pausing work on the standalone apps.**

### Step 1 — Close the schema-shaping decisions (~1 hour, no code)

- [ ] **D4** ID strategy
- [ ] **D5** Module separation
- [ ] **D9** Definition of `customer`
- [ ] **D10** Soft delete or hard delete
- [ ] **D6 / D7** Grid and component system — cheap to decide, prevents app #3 adding a third variant

Record each with a one-line rationale.

**D2 and D3** are urgent but tied to the API — answer them before the first Fastify route, not before the schema work above.

### Step 2 — Write the glossary (~1 hour)

Start from the term list in 1.5. Resolve `customer` explicitly, including what it is *not*.

### Step 3 — Shared types package (~1–2 hours)

Generate, publish as `@logistics/types`, consume from rates-app, add a regeneration script so it doesn't drift.

### Step 4 — Service pattern on one feature (~half day)

**Do not convert rates-app.** Pick the feature you're actively working on. Create `src/services/`, move its queries, write `ServiceError` and one DTO mapper, point the components at the service.

The goal is a working reference implementation, not coverage. Once one exists, 1.1 stops being a project and becomes a habit.

### Step 5 — TypeScript on-ramp (~30 min)

`tsconfig.json` with `allowJs: true`, `strict: true`, `noEmit: true`. Next new file is `.ts`. Convert nothing.

---

# Part 6 — Ongoing Habits

Rules that apply from Step 4 onward.

- No `supabase.from()` outside `src/services/`
- New files in rates-app are TypeScript
- New tables follow Step 1 conventions and carry audit columns
- New domain terms go in the glossary the day they're invented
- New apps use the D6/D7 stack and get a subdomain
- Any code needing a map calls GeoBrain, never HERE or OSM directly

---

# Part 7 — Triggers

Tier 2 items shouldn't sit on a calendar. Each has a natural trigger.

| When this happens | Do this |
|---|---|
| Something needs data without a browser — cron, webhook, export | Stand up `logistics-api`. This is the real signal. |
| About to write the first Fastify route | D2 and D3 must be answered first |
| Starting Inbound | Design the event log and projection first, before any UI |
| stufferPlanner needs to survive a refresh | Give it a schema |
| A second app needs the same component | Extract `@logistics/ui` |
| About to create app #3 | It gets a subdomain and uses the D6/D7 stack |
| About to build GeoBrain | D11 and D12 must be answered first |
| Starting to write AI tools | Stop — `logistics-api` must exist first |

---

# What Success Looks Like

Consolidation should eventually be:

- Swap service bodies from Supabase calls to `fetch` — components untouched
- Point three apps at a shared types package they already use
- Move the hub from links to routes

That is a week of work. Skipping Steps 1–4 turns the same job into a rewrite.
