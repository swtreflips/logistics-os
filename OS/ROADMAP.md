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

| | Schedules | rates-app | stufferPlanner | geoapi | logistics-api |
|---|---|---|---|---|---|
| Role | Schedules module | Rates module | Planner module | GeoBrain service | Business Services API |
| Stack | Vite + React | Vite + React 18 | Vite + React 19 | Next.js 16 | Fastify |
| Language | TypeScript | JavaScript | TypeScript | TypeScript | — |
| UI | custom | MUI + MUI X DataGrid | Radix + ag-Grid | — | — |
| Data layer | `lib/supabase.ts` | `features/*/services/` | `data/repos/` | own routes | — |
| Backend | Supabase + RPCs | Supabase | Supabase, 6 `planner_` tables | Supabase cache, HERE, Nominatim | — |
| Deploy | Vercel | Vercel | Vercel | Vercel — **live** | Railway (not built) |

**Shared:** one Supabase project, one auth model, RLS on **30 of 30** public tables, hub of cards for internal navigation.

Inbound is an early `mapLibre` spike, not yet a module. The Python carrier scrapers feed Schedules and are deliberately separate.

`geoapi` remains the reference: a real backend service with authenticated routes, a cache layer, and no business logic.

## What Was Wrong Here Before

This section used to say the real problem was that **rates-app queries Supabase directly from the browser, so no service layer exists anywhere.** That was measured and is false:

- `rates-app` — six service modules under `features/*/services/`
- `Schedules` — `findNearbySchedules()`, `getDistinctPOLs()`, over Postgres RPCs
- `stufferPlanner` — repository interfaces with two implementations

Across all three, **zero business queries are issued from a React component.** Tier 1.1 below — *"the most important item here"* — was reached without anyone running it as a project.

Correcting this matters because a roadmap that misdescribes the present sends effort at problems that are already solved.

## The Real Problem

Two things, and they are different.

### 1. The data layer runs in the browser

The seam exists, but it lives inside three React bundles. So:

- A cron job, webhook, or export has nowhere to call
- LangGraph cannot call a React bundle — every AI phase is hard-blocked
- Nothing outside a signed-in browser session can reach a capability

This is the P2 gate, and it is now a **relocation**, not a rewrite. Part 5 describes the move.

### 2. Business rules still live in components

Rule 3 is met. **Rule 2 is not.** Putting a query behind a function does not put the *rule* behind it, and the rules stayed in React:

| App | Rules still in components |
|---|---|
| stufferPlanner | who may edit a row, which rows are visible, container eligibility, allocation capacity |
| rates-app | rate validity windows, ranking, submission gating |
| Schedules | grouping and sort order in `state/` |

This is the gap that produces bugs *today*, not at P2. A worked example, from July 2026: the planner's grid decided which PO lines a supplier could see and edit, using the user's single primary organization. The database had already scoped the query correctly and returned 34 rows for a two-plant supplier; the component discarded 9 of them and made the rest read-only. No error, no failing test, no type error — the rule in React silently contradicted the rule in Postgres, and it took rendering the app as that user to notice.

That is precisely the failure `ARCHITECTURE.md` Part 6 predicts. The rule had been written down long before the bug, and being written down did not help, because it was not enforceable anywhere.

**A rule that exists in two places has already diverged; you just have not looked yet.**

---

# Part 3 — Decisions Log

| # | Decision | Status |
|---|---|---|
| D1 | API layer | **closed** — standalone Fastify service on Railway |
| D2 | Authorization: RLS-authoritative or service-authoritative | open — urgent |
| D3 | Service identity for jobs and AI agents | open — urgent |
| D4 | ID strategy: uuid or bigint | **closed by practice** — `uuid` / `gen_random_uuid()` |
| D5 | Module separation: Postgres schemas or table prefixes | **closed by practice** — table prefixes |
| D6 | Standard grid library | open |
| D7 | Standard component system: MUI or Radix | open — **the visual system half is closed**, see `DESIGN.md` |
| D14 | Per-module accent, or one accent everywhere | **closed** — per-module, assigned in `DESIGN.md` |
| D8 | Domain strategy | **closed** — subdomains per module, `api.domain.com` |
| D9 | Definition of `customer` | open — see `GLOSSARY.md` |
| D10 | Soft delete or hard delete | open |
| D11 | GeoBrain consumers: API only, frontends too, or both | open — service is live, frontends already connected |
| D12 | GeoBrain cache store: shared Supabase schema or dedicated | open — currently shared Supabase |
| D13 | Name for the data-access layer | open — three apps use three names |

## D4 and D5 — Closed By Practice

Both were decided in code before they were decided on paper. Recording them stops a future app re-litigating a question the schema has already answered.

**D4 — `uuid`, generated with `gen_random_uuid()`.** Every table created for the planner and drayage modules uses it. A handful of older reference tables key on natural codes instead, which is correct for them and not a counter-example.

**D5 — table prefixes, not Postgres schemas.** `planner_` (6 tables), `drayage_` (5), `sched_`. Genuinely shared entities are deliberately unprefixed: `organizations`, `profiles`, `notifications`, `organization_groups`. The `rates` and `schedules` tables predate the convention and are not worth renaming.

The rule going forward: **module-private tables take a prefix; shared entities do not.**

One consequence worth naming — `geocode_cache` sits unprefixed among business tables. It is neither shared business data nor module-private, and it is the table D12 is about.

## D13 — What To Call The Data Layer

`rates-app` says `services/`, Schedules says `lib/`, stufferPlanner says `repos/`. Same seam, three dialects.

Low cost today, real cost at P2: each is a separate translation when capabilities move to Fastify. Pick one for the next app and leave the existing three alone.

`services/` is the term used throughout `ARCHITECTURE.md`, which is an argument for it. Against: stufferPlanner's `repos/` are genuinely repositories — data access with no business rules — and `ARCHITECTURE.md` Part 2 separates those concepts deliberately. Both layers eventually exist; the browser currently has only one of them.

## D2 — Authorization

Currently RLS does everything, because the browser talks to Postgres directly. Once Fastify sits in between:

- **User JWT forwarded** — RLS stays authoritative, API adds business rules. Least disruption, but jobs and agents have no JWT.
- **Service role key** — API becomes authoritative, RLS bypassed server-side. More power, but every rule RLS gives you free must be reimplemented and tested.

Most systems land on: user JWT for interactive requests, service role for jobs and agents, services enforcing rules in both paths.

**Decide before the first Fastify route.**

**This is a security decision, not only an architectural one.** Cross-party isolation is currently verified and holds because RLS sits in the request path. Choosing the service role key bypasses RLS server-side and moves that guarantee into service code that does not exist yet — see `SECURITY.md` Part 7. D3 is the same decision for machine identities.

## D11 and D12 — GeoBrain

**GeoBrain is built and deployed** — the `geoapi-next` repo, live at `geoapi-next.vercel.app`, Next.js on Vercel as specified. Six routes: `geocode`, `route`, `route-batch`, `within`, `within-batch`, `healthz`. It uses Supabase for its cache and calls HERE and Nominatim upstream.

So these are decisions about a **running service**, not a future one, and both are partly answered in practice.

**D11 — consumers.** Currently two: signed-in rates-app browser sessions presenting a Supabase JWT, and server-side scripts presenting a separate `GEO_SERVICE_TOKEN`. The frontend-direct path is already open, so the reversible choice has been spent. What remains is whether the Business Services API becomes a third consumer, or the only one, once it exists.

**D12 — cache store.** Currently shared Supabase. Still needs the schema separation from D5: geocode and route caches will grow to millions of rows beside business tables and must be excluded from the backup, audit, and RLS conventions that exist for business data.

Security posture is sound — every billable route authenticates, the anon key is explicitly rejected as a caller credential, CORS fails closed in production, and batch sizes are capped. Open items are per-caller rate limiting and a billing alert on the HERE account. See `SECURITY.md`.

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

### 1.1 Keep queries out of components — ✅ done, hold the line

This was *"the most important item here."* It is met in all three apps: `features/*/services/`, `lib/supabase.ts`, `data/repos/`. The only direct calls outside the layer are two identity lookups in `rates-app`'s `AuthProvider`, which is shell code, not domain logic.

What remains of the original intent:

- ✅ No `supabase.from()` outside the data layer
- ⬜ **The layer returns DTOs, never raw Postgres rows** — partly true; stufferPlanner's repos map to domain types, the other two return rows largely as-is
- ⬜ **Signatures use domain types, never Supabase client types**

The rule stays in `CLAUDE.md` to prevent regression, not to describe a cleanup.

### 1.1b Move business rules out of components — **the actual most important item**

Rule 3 is met; Rule 2 is not. See Part 2, *The Real Problem*, for what this costs and the July 2026 example of what it costs *silently*.

The test for whether a line belongs in the data layer: **would it still be true if the screen were rendered differently?** "Which rows may this user edit" is true regardless of grid library. "Which column is sorted" is not.

Convert opportunistically, the same way 1.1 succeeded. Start with permission decisions — they are the ones that fail invisibly, because a wrong answer looks like a working screen.

### 1.2 Generate and share database types

```bash
supabase gen types typescript --project-id <id> > packages/types/src/db.ts
```

Highest leverage per hour on this list. Publish as `@logistics/types` via a `file:` dependency — no monorepo tooling required.

### 1.3 Freeze schema conventions — partly settled, one real gap

Already true in practice: `uuid` IDs (D4) · `snake_case`, plural tables, `<entity>_id` foreign keys · `timestamptz` UTC · module prefixes (D5) · **RLS on 30 of 30 tables**.

Still open: soft-delete policy (D10).

**The gap: `created_by` / `updated_by` exist on zero of 30 tables.**

This is the largest unmet item in these documents and the only one that gets worse by waiting — every day of writes is history that cannot be recovered later. `created_at` is on roughly half.

`ARCHITECTURE.md` Part 5 requires who / when / old / new / reason / AI-involved / approver on every consequential action. None of that is being captured.

One partial exception, and it points the way: `planner_po_line_events` records field-level changes for cargo ready and CBM via an AFTER UPDATE trigger. That is the pattern — a trigger, so a CSV upload and an inline edit produce identical history with no app-side cooperation. It covers two fields on one table.

### 1.4 Answer D2 and D3

See Part 3.

### 1.5 Write a glossary — ✅ done

`GLOSSARY.md`. Grounded in the schema rather than invented, so it describes the terms the tables
actually use.

Two terms are deliberately left **undefined**: `customer` (D9) and `shipment`. Both are owned by
Inbound, which does not exist yet, and an invented definition now would be adopted by three
modules before the module that owns it gets a say.

It also records three naming inconsistencies already present — `pol`/`port_of_loading`,
`pod`/`port_of_discharge`, `transit_days`/`transit_time_days` — to be resolved opportunistically,
not migrated.

## Tier 2 — Soon

**2.1 Stand up `logistics-api`.** The change that unlocks AI, jobs, and cross-module queries. 1.1 *is* done consistently, so this is mechanical — relocation, not rewrite.

Consequences to plan for: cross-origin calls need CORS and a token strategy (D2); two deploy targets means API changes must stay backward-compatible; the API needs its own repo, env management, and server-side Supabase client. `geoapi` has already solved the CORS and token-strategy problems once — reuse that shape rather than rediscovering it.

**2.2 Give stufferPlanner persistence.** ✅ **Done, July 2026.** Six `planner_` tables with RLS, an append-only event log for cargo ready and CBM, sibling-organization grouping so a multi-plant supplier sees all its plants, and repository interfaces with Local and Supabase implementations behind `VITE_DATA_SOURCE`.

The repository seam earned itself immediately: swapping sample data for the real database changed no consuming component. Worth noting against Part 5's advice not to build repositories in the browser — see the note there.

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

# Part 5 — Adopting The Service Layer

The apps were started with browser-direct Supabase queries. That was the straightforward way to begin and it is not a mistake to correct all at once.

Two rules replace a migration project:

> **New code always goes through a service. No exceptions, starting now.**
>
> **Existing code converts when you happen to touch it.** Never as a dedicated effort.

The first rule stops the debt growing. The second amortizes cleanup across normal work. There is no week where you ship nothing.

## Do Not Filter By AI Usefulness

A tempting shortcut is to convert only the queries that look like they will matter to the assistant. Don't.

You cannot predict which capability the AI will want. `getForwarderWinRate` sounds AI-relevant; `getLaneById` sounds boring, and the assistant will call the boring one constantly.

Worse, the filter re-introduces AI reasoning into human app design, which is precisely what `README.md` — *The Foundation* — says not to do.

**Every data operation goes through a service because that is how the app should be built.** AI-readiness is the byproduct, never the justification.

## Build These Three Things Once

```
src/services/
  _client.ts      the ONLY place createClient() is called
  errors.ts       ServiceError, NotFoundError, PermissionError
  rateService.ts  the first real service
```

`errors.ts` carries more weight than it appears — it is where the permission rule from `ARCHITECTURE.md` Part 7 actually lives:

```ts
export class ServiceError extends Error {
  constructor(code, message, cause) {
    super(message);
    this.name = 'ServiceError';
    this.code = code;
    this.cause = cause;
  }
}

// Deliberately identical to NotFound at the boundary —
// a denial must not reveal that the record exists.
export class PermissionError extends ServiceError {
  constructor() { super('NOT_FOUND', 'Not found'); }
}
```

Write new service files as `.ts` even in `rates-app`. Services are where types pay for themselves, because these signatures become the API contract later.

## The Conversion

A realistic page today:

```jsx
// BEFORE
useEffect(() => {
  (async () => {
    const { data } = await supabase
      .from('rates').select('*, forwarders(name)').eq('lane_id', laneId);

    const active = data.filter(r => new Date(r.valid_until) >= new Date());
    const sorted = active.sort((a, b) => a.total_cost - b.total_cost);
    setRows(sorted.map(r => ({ ...r, forwarder: r.forwarders.name })));
  })();
}, [laneId]);
```

The two middle lines are the point. *"Which rates are still valid"* and *"best rate first"* are **business rules**, sitting in a `useEffect`. They exist only on this page, so every other page, report, and future tool is free to answer the same question differently.

```ts
// AFTER — src/services/rateService.ts
export async function getActiveRatesForLane(laneId: string): Promise<Rate[]> {
  const { data, error } = await supabase
    .from('rates')
    .select('id, lane_id, total_cost, currency, valid_from, valid_until, forwarders(id, name)')
    .eq('lane_id', laneId);

  if (error) throw new ServiceError('RATES_LOOKUP_FAILED', 'Could not load rates', error);

  return data
    .filter(isCurrentlyValid)
    .sort(byTotalCostAscending)
    .map(toRateDTO);
}
```

```jsx
// The component becomes boring, which is the goal
useEffect(() => {
  getActiveRatesForLane(laneId).then(setRates);
}, [laneId]);
```

## The Payoff

When `logistics-api` exists, only the service body changes:

```ts
export async function getActiveRatesForLane(laneId: string): Promise<Rate[]> {
  const res = await fetch(`${API}/lanes/${laneId}/rates?active=true`);
  if (!res.ok) throw new ServiceError('RATES_LOOKUP_FAILED', 'Could not load rates');
  return res.json();
}
```

**No component changes.** The validity and ranking rules move server-side, where the AI tool calls the same code — so the assistant's idea of "the best current rate" is guaranteed to match what the screen shows.

That guarantee is the entire reason for this work.

## What To Convert First

The screen you are already working on this week. Not the biggest, not the most AI-relevant, not the messiest.

The first conversion's real job is to establish `_client`, `errors`, and one DTO mapper, so the second one takes twenty minutes.

## Three Ways To Over-Engineer This

**Don't build a repository layer in the browser** — *with one demonstrated exception.*

Services calling Supabase directly is correct for most cases, and the repository split normally happens when the code moves to Fastify.

stufferPlanner did it anyway, and was right to. It had sample data and no schema, so the interface was doing real work immediately: two implementations, `Local` and `Supabase`, selected by `VITE_DATA_SOURCE`. When the database arrived, swapping them changed no consuming component, and each repo could be migrated and verified on its own rather than flipping five at once and debugging auth, RLS and column mapping simultaneously.

**The distinction: build the interface when a second implementation exists today.** Not when one might exist later. "Supabase now, Fastify eventually" does not qualify — that is one implementation and a plan.

**Keep DTO mappers dumb.** Rename fields, drop internals, flatten a join. No validation frameworks, no class hierarchies.

**Don't convert everything.** A file you are not touching is fine as-is, indefinitely.

---

# Part 6 — Next Steps

Ordered. Early steps are decisions and documents that cost no code and stop further divergence. Code work is last and deliberately small.

**None of this requires pausing work on the standalone apps.**

### Step 1 — Close the schema-shaping decisions (~1 hour, no code)

- [x] **D4** ID strategy — `uuid`, settled by practice
- [x] **D5** Module separation — table prefixes, settled by practice
- [x] **Party isolation model** — `organization_id` on every external-facing table, RLS keyed on `my_org()` / `my_orgs()` / `my_org_type()`, sibling plants grouped via `organizations.group_id`. Verified across internal, forwarder and multi-plant supplier accounts. **This was the most consequential item on the list and it is done.**
- [ ] **D9** Definition of `customer` — see `GLOSSARY.md`
- [ ] **D10** Soft delete or hard delete
- [ ] **D6 / D7** Grid and component system — cheap to decide, prevents app #4 adding a third variant
- [ ] **D13** One name for the data layer

Record each with a one-line rationale.

**D2 and D3** are urgent but tied to the API — answer them before the first Fastify route, not before the schema work above.

### Step 1b — Add audit columns (~half day, and it decays)

`created_by` / `updated_by` are on zero of 30 tables. Every day of writes is history that cannot be reconstructed. This has overtaken everything else in Tier 1 by urgency, because it is the only item whose cost grows while it waits.

Start with the tables that already carry consequential writes: `planner_po_lines`, `planner_containers`, `planner_allocations`, `rates`, `rate_submissions`.

### Step 2 — ✅ done — `GLOSSARY.md` exists

Resolve `customer` (D9) into it when Inbound's data model is drafted, including what it is *not*.

### Step 3 — Shared types package (~1–2 hours)

Generate, publish as `@logistics/types`, consume from rates-app, add a regeneration script so it doesn't drift.

### Step 4 — Move one *rule* into the data layer (~half day)

The original Step 4 was "service pattern on one feature." That pattern now exists in three apps; the queries have moved. **The rules have not.**

Pick one permission decision currently living in a component and move it down. The reference candidate is stufferPlanner's `canEditRow` — a rule about who may edit which PO line, sitting in a grid column definition, which has already been observed disagreeing with the database.

Write `ServiceError` / `PermissionError` while you are there — Part 5 shows why the permission-denial shape belongs in shared code rather than in each caller.

The goal is one worked example, not coverage. Once it exists, 1.1b stops being a project and becomes a habit — exactly how 1.1 got done.

### Step 5 — TypeScript on-ramp (~30 min)

`tsconfig.json` with `allowJs: true`, `strict: true`, `noEmit: true`. Next new file is `.ts`. Convert nothing.

---

# Part 7 — Ongoing Habits

Rules that apply from Step 4 onward.

- No `supabase.from()` outside `src/services/`
- New files in rates-app are TypeScript
- New tables follow Step 1 conventions and carry audit columns
- New domain terms go in the glossary the day they're invented
- New apps use the D6/D7 stack and get a subdomain
- Any code needing a map calls GeoBrain, never HERE or OSM directly

---

# Part 8 — Triggers

Tier 2 items shouldn't sit on a calendar. Each has a natural trigger.

| When this happens | Do this |
|---|---|
| Something needs data without a browser — cron, webhook, export | Stand up `logistics-api`. This is the real signal. |
| About to write the first Fastify route | D2 and D3 must be answered first |
| Starting Inbound in earnest | Design the event log and projection first, before any UI. `mapLibre` is a spike, not the module. |
| ~~stufferPlanner needs to survive a refresh~~ | ~~Give it a schema~~ — **done, July 2026** |
| Touching a component that decides who may see or do something | Move the rule into the data layer (1.1b) |
| Creating any new table | It carries `created_by` / `updated_by` — no exceptions (1.3) |
| A second app needs the same component | Extract `@logistics/ui` |
| About to create app #4 | It gets a subdomain, the D6/D7 stack, and the D13 layer name |
| Before the first external party logs in | `SECURITY.md` Part 5 gate — self-registration is currently **enabled** |
| `logistics-api` starts calling GeoBrain | Settle D11 — is it the only consumer, or a third one? |
| GeoBrain cache growth becomes visible | Settle D12 — schema separation per D5 |
| Starting to write AI tools | Stop — `logistics-api` must exist first |

---

# What Success Looks Like

Consolidation should eventually be:

- Swap service bodies from Supabase calls to `fetch` — components untouched
- Point three apps at a shared types package they already use
- Move the hub from links to routes

That is a week of work. Skipping Steps 1–4 turns the same job into a rewrite.
