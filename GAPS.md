# GAPS.md

# Consolidation Groundwork

Adjustments to make **while still building standalone apps**, so that consolidating them into one Logistics OS later is a refactor rather than a rewrite.

This is not a plan to merge the apps. It is a list of habits and decisions that keep the merge cheap.

---

# The Principle

The cost of a gap is not its size. It is how much it **compounds**.

| Compounds badly | Doesn't compound |
|---|---|
| Schema and naming decisions | Which grid library |
| Direct-DB-access spreading through components | Which CSS framework |
| Identity and tenancy model | Monorepo vs. polyrepo |
| Domain vocabulary drift | React version |
| Missing audit columns | Folder layout |

The right-hand column looks like the mess. The left-hand column is what actually determines whether consolidation takes two weeks or two quarters.

**Fix the left column now. Freeze the right column and converge it later.**

---

# Current State

| | rates-app | stufferPlanner | logistics-os |
|---|---|---|---|
| Stack | Vite + React 18 | Vite + React 19 | none (docs only) |
| Language | JavaScript | TypeScript | — |
| UI | MUI + MUI X DataGrid | Radix + ag-Grid | — |
| Styling | Tailwind 3 | Tailwind 4 | — |
| State | — | zustand | — |
| Data | Supabase, **direct from browser** | none — CSV + client state | — |
| Deploy | Vercel | Vercel | — |

**Shared:** one Supabase project, one auth model, RLS working for internal/external.

That shared foundation is the hard part, and it already exists. Everything below is recoverable.

---

# Tier 1 — Do Now

Cheap today. Expensive or irreversible later.

## 1.1 Stop writing `supabase.from()` inside React components

**This is the single most important item in this document.**

Every new browser-direct query is another line that has to be found and rewritten when the API layer lands, and another business rule the AI cannot reach.

You do not need to build the backend yet. You need to **shape the call sites** so the backend can slide in underneath without touching the UI.

Today:

```js
// inside a component — don't add more of these
const { data } = await supabase
  .from('rates')
  .select('*')
  .eq('lane_id', laneId);
```

Instead:

```js
// src/services/rateService.js
// Still calls Supabase from the browser today.
// The signature is what matters — it does not leak Supabase types.
export async function getRatesForLane(laneId) {
  const { data, error } = await supabase
    .from('rates')
    .select('*')
    .eq('lane_id', laneId);
  if (error) throw new ServiceError('RATES_LOOKUP_FAILED', error);
  return data.map(toRateDTO);
}
```

The component calls `getRatesForLane(laneId)` and knows nothing else.

When the API layer arrives, the body becomes `fetch('/api/rates?lane=' + laneId)` and **no component changes**. Same when a LangGraph tool needs it — it imports the service, not the query.

Rules:
- No `supabase.from()` outside `src/services/`
- Services return DTOs, never raw Postgres rows
- Service signatures use domain types, never Supabase client types
- Existing violations: leave them. Convert opportunistically when you touch a file.

## 1.2 Generate and share database types

```bash
supabase gen types typescript --project-id <id> > packages/types/src/db.ts
```

Highest leverage-per-hour item on this list. One generated file, committed, consumed by every app. Prevents the classic consolidation discovery that two apps disagree about what a column contains.

Publish as `@logistics/types` (a plain folder + `file:` dependency is fine — no monorepo tooling required).

## 1.3 Freeze the schema conventions

Write these down once and apply to every new table in every app:

- **ID strategy** — `uuid` vs `bigint identity`, and stick to it
- **Timestamps** — `created_at`, `updated_at`, always `timestamptz`, always UTC
- **Audit columns** — `created_by`, `updated_by` on every business table
- **Soft delete** — `deleted_at` vs. hard delete, decided once
- **Naming** — `snake_case`, plural tables, `<entity>_id` foreign keys
- **Module ownership** — Postgres schemas (`rates.`, `stuffer.`, `inbound.`) or table prefixes, so it stays obvious who owns what in a shared database

Retrofitting audit columns loses all history prior to the change. Adding them now is free.

## 1.4 Decide the identity and authorization model — and write it down

Currently RLS does all authorization, because the browser talks to Postgres directly. That works today and breaks the moment anything non-interactive needs data.

Answer these explicitly:

- What does RLS key off — `auth.uid()`, an `org_id` claim, a role claim?
- When the API layer exists, does it use the **user's JWT** (RLS stays authoritative) or the **service role key** (services become authoritative)?
- **What identity do background jobs and AI agents run as?** `DESIGN.md` says every tool executes as the authenticated user. Scheduled jobs and Phase 5 proactive monitoring have no user. You need a service principal concept, and its permissions need defining.

Pick the answers now. They shape every service method you write between now and then.

## 1.5 Agree on domain vocabulary

If rates says `lane`, stuffer says `route`, and inbound says `shipment_leg` for the same concept, consolidation becomes a translation project.

Start `GLOSSARY.md`. One line per term. Non-negotiable candidates:

`lane` · `route` · `leg` · `shipment` · `booking` · `container` · `customer` · `supplier` · `forwarder` · `carrier` · `port` · `origin` / `destination` · `ETA` / `ETD` / `ATA` / `ATD` · `cargo ready` · `item` / `SKU`

**`customer` is the one that will bite.** For an inbound importer it is genuinely ambiguous — end retailer, DC, or delivery recipient. Half the planned AI queries depend on it. Define it before three apps define it differently.

This costs an hour and it is the cheapest item here.

## 1.6 Stop adding JavaScript to rates-app

Vite supports incremental TypeScript. Don't convert the existing app — just add `tsconfig.json` with `allowJs: true` and write **new** files as `.ts` / `.tsx`.

Every JS file written today is a file that must be typed later before it can be safely shared. `CLAUDE.md` already mandates "TypeScript everywhere, no `any`" — right now the docs describe a codebase that doesn't exist.

## 1.7 Settle the domain strategy before adding apps

Session sharing across the hub depends on this and it is painful to change once users have bookmarks and links:

- `rates.company.com` + `stuffer.company.com` → cookies shareable across subdomains, single sign-on across cards works naturally
- `rates-app.vercel.app` + `stuffer.vercel.app` → separate origins, users re-authenticate per card

Decide before app #3.

---

# Tier 2 — Do Soon

Needed before real consolidation. Not urgent this week.

## 2.1 Extract the API layer

The one change that unlocks AI, background jobs, and cross-module queries. If Tier 1.1 is done consistently, this is mechanical: swap service bodies from Supabase calls to `fetch`.

Options, in rough order of fit:
- **Supabase Edge Functions** — closest to what exists, no new hosting, keeps RLS natural
- **Next.js API routes** — what the docs specify; means a Next app in front of, or alongside, the SPAs
- **Small Fastify service on Vercel** — most control, most setup

`CLAUDE.md` lists this as undecided. Decide it when you do it, not before.

## 2.2 Give stufferPlanner persistence

It has no Supabase dependency at all — CSV in, client state, CSV out. It cannot participate in cross-module anything until it has a schema in the shared database.

Note this is not a consolidation cost. It's unfinished work that exists regardless.

## 2.3 Shared UI package

Once two apps need the same thing (auth guard, org switcher, error boundary, empty state, hub nav), extract `@logistics/ui`. Not before — premature extraction of a component library is its own trap.

## 2.4 Event log for Inbound

Inbound hasn't started, so build it right the first time: events table, current-state projection maintained on insert, no mutable status column.

Note the tension with `CLAUDE.md`'s "never duplicate business state" — a projection table *is* duplicated state, and you need one to hit the sub-200ms goal and to filter "all delayed containers" in SQL. The rule needs amending: **derive state from events, materialize it for reads, never let anything write the projection directly.**

---

# Tier 3 — Explicitly Defer

Not doing these is correct right now. Listed so they don't feel like oversights.

- **Monorepo migration** — shared packages via `file:` deps get 90% of the benefit at 10% of the cost
- **Frontend stack convergence** — freeze new divergence, converge opportunistically; two grid libraries is ugly, not dangerous
- **Next.js migration** — Vite SPA + separate API service is a perfectly good architecture; don't migrate for the sake of the doc
- **Sentry / OpenTelemetry** — add when there's production traffic worth observing
- **Anything LangGraph** — blocked on 2.1 regardless; writing tools before services exist means writing them twice

---

# Documentation Fixes

The current docs describe a system that doesn't match either app. Fix so they stop misleading:

- `CLAUDE.md` mandates **Next.js** — both apps are Vite SPAs
- `CLAUDE.md` mandates **MUI DataGrid** — stufferPlanner uses ag-Grid
- `CLAUDE.md` mandates **TypeScript everywhere** — rates-app is entirely JavaScript
- `CLAUDE.md` and `DESIGN.md` overlap heavily; CLAUDE.md loads into every AI context window, so it should hold *enforceable rules* (conventions, commands, layout) and leave philosophy in DESIGN.md
- **Missing:** `SCHEMA.md`, `EVENTS.md`, `GLOSSARY.md` — the actual domain content
- `rates-app` has ~30 markdown files including `NEXTSTEPS.md` **and** `NEXTSTEPSV2.md`, plus three separate assessment documents. Consolidate; contradictory docs are worse than none.

---

# Decisions Log

Answer these and record them here. Each one is currently unstated and load-bearing.

| # | Decision | Status |
|---|---|---|
| D1 | API layer: Edge Functions / Next.js routes / Fastify | open |
| D2 | Authorization: RLS-authoritative or service-authoritative | open |
| D3 | Service identity for jobs and AI agents | open |
| D4 | ID strategy: uuid or bigint | open |
| D5 | Module separation: Postgres schemas or table prefixes | open |
| D6 | Standard grid library | open |
| D7 | Standard component system: MUI or Radix | open |
| D8 | Domain strategy: subdomains or separate origins | open |
| D9 | Definition of `customer` | open |
| D10 | Soft delete or hard delete | open |

---

# Next Steps

Ordered. The early steps are decisions and documents that cost no code and stop further divergence; the code work is deliberately last and deliberately small.

Nothing here requires pausing work on the standalone apps.

---

## Step 0 — Initialize the repo (5 min)

```bash
cd logistics-os && git init && git add . && git commit -m "Architecture docs and consolidation groundwork"
```

`logistics-os` isn't under version control yet.

---

## Step 1 — Close the schema-shaping decisions (~1 hour, no code)

These five shape every table you create from now on. Answering them late means migrating tables that already have data.

- [ ] **D4** ID strategy — `uuid` or `bigint identity`
- [ ] **D5** Module separation — Postgres schemas or table prefixes
- [ ] **D9** Definition of `customer` — the one that will bite
- [ ] **D10** Soft delete or hard delete
- [ ] **D8** Domain strategy — subdomains or separate origins

Record the answers in the Decisions Log above with a one-line rationale each.

**Defer D1, D2, D3** (API layer, authorization model, service identity) until you actually build the API. They're load-bearing but they don't constrain today's work.

**Decide D6 and D7** (grid library, component system) in the same sitting if you can — they cost nothing to decide and prevent app #3 from adding a third variant.

---

## Step 2 — Write GLOSSARY.md (~1 hour)

Start from the term list in 1.5. One line per term, plus the fields that identify it.

Resolve `customer` explicitly and write down what it is *not*. Everything downstream — customer-facing queries, delay notifications, the AI's "which customers are affected" flow — inherits that definition.

---

## Step 3 — Stand up the shared types package (~1–2 hours)

```bash
mkdir -p packages/types/src
supabase gen types typescript --project-id <id> > packages/types/src/db.ts
```

Add a minimal `package.json` to `packages/types`, then consume it from rates-app via a `file:` dependency. No monorepo tooling needed.

Add a regeneration script so it doesn't drift:

```json
"scripts": { "types:gen": "supabase gen types typescript --project-id <id> > ../types/src/db.ts" }
```

---

## Step 4 — Establish the service pattern on one feature (~half day)

**Do not convert rates-app.** Pick the single feature you're actively working on and do only that one:

1. Create `src/services/`
2. Move that feature's queries into a service module (pattern in 1.1)
3. Write the small shared pieces once — `ServiceError`, one `toXDTO` mapper
4. Point the components at the service

The goal is a working reference implementation, not coverage. Once one exists, 1.1 stops being a project and becomes a habit: new features get services, touched files get converted, everything else waits.

---

## Step 5 — Open the TypeScript on-ramp (~30 min)

Add `tsconfig.json` to rates-app with `allowJs: true`, `strict: true`, `noEmit: true`. Write the next new file as `.ts`. Convert nothing.

---

## Then: Ongoing Habits

Not tasks — rules that apply from Step 4 onward.

- No `supabase.from()` outside `src/services/`
- New files in rates-app are TypeScript
- New tables follow the Step 1 conventions and carry audit columns
- New domain terms go in `GLOSSARY.md` the day they're invented
- New apps use the D6/D7 stack

---

## Then: Trigger-Based, Not Date-Based

Tier 2 items shouldn't sit on a calendar. Each has a natural trigger:

| When this happens | Do this |
|---|---|
| Something needs data without a browser — a cron job, a webhook, an export | Build the API layer (2.1). This is the real signal. |
| You start building Inbound | Design the event log and projection first (2.4), before any UI |
| stufferPlanner needs to survive a refresh or be seen by another user | Give it a schema (2.2) |
| The second app needs the same component | Extract `@logistics/ui` (2.3) |
| You're about to create app #3 | D8 must be locked, and it uses the D6/D7 stack |
| You start writing AI tools | Stop — 2.1 must exist first, or you'll write every tool twice |

---

## What Success Looks Like

Six months from now, consolidation should be:

- Swap service bodies from Supabase calls to `fetch` — components untouched
- Point three apps at one shared types package they already use
- Move the hub from links to routes

That is a week of work. Skipping Steps 1–4 turns the same job into a rewrite.

---

# Guiding Questions

Before merging anything new, ask:

> **Could a background job call this without a browser?**

If no, the logic is in the wrong layer.

> **If this table existed in another module, would it be named the same way?**

If no, fix the name now.

> **Is this divergence in the data or in the pixels?**

Data divergence is debt. Pixel divergence is a preference.

---

*Assessment based on `package.json`, repository layout, and the existing markdown docs across `logistics-os`, `rates-app`, and `stufferPlanner`. Application source was not read in depth — specifics in 1.1 may need adjusting to match actual call-site patterns in `rates-app/src`.*
