# Modules

The Logistics OS is composed of independent but interconnected modules. Each owns a portion of the logistics lifecycle while exposing reusable capabilities through backend services.

Modules are **not isolated applications**. Together they form one operational platform feeding one shared data model.

---

# Lifecycle

```
Planning
   │
Schedules  ──▶  Rates  ──▶  Container Planning
                                   │
                                   ▼
                         Shipment Execution
                                   │
                                   ▼
                          Inbound Operations
                                   │
                                   ▼
                       AI Operations Assistant
```

Planning creates shipments. Execution tracks shipments. AI understands shipments.

---

# Business Modules vs. Platform Services

**Business modules** own a domain. Each has a lifecycle stage, a user, and its own operational meaning.

**Platform services** are shared capabilities with no business domain. Modules consume them; they never own logistics state.

> **The test:** if a capability would still make sense at a company that isn't in logistics, it is a platform service, not a module.

GeoBrain passes that test. Inbound does not.

Platform services do not appear in the lifecycle and do not get their own AI phase. They make modules possible.

---

# Module 1 — Schedules

## Purpose

Help coordinators determine the best ocean freight routing before booking. Aggregates carrier schedules into one searchable interface instead of checking carrier websites manually.

## Responsibilities

Collect schedules from carriers · compare transit times · compare transshipment ports · identify direct sailings · track schedule changes · present routing alternatives

## Data Sources

Carrier websites · carrier APIs · Python scraping services

## Status — built

A React dashboard over schedule data, reading through named capability functions backed by Postgres RPCs (`nearby_schedules`, `distinct_pols`). Geospatial lookups go to GeoBrain, which replaced a Render-hosted FastAPI geocoder in July 2026 and is faster than it was.

The Python scrapers that populate `schedules` are deliberately kept outside the platform repos for now.

## AI Usage

*What's the fastest routing? Which carrier has the fewest transshipments? Which services arrive before August 10? Compare ONE and Maersk for this lane.*

The AI recommends. Humans decide.

---

# Module 2 — Rates

## Purpose

A collaborative rate management platform between the company and freight forwarders. Replaces spreadsheets and email with structured data, and becomes the source of truth for quoted freight rates.

## Responsibilities

Receive quotes · store historical rates · compare forwarders · track validity periods · manage lane pricing · maintain quoting history

## AI Usage

*Who quoted the best rate? How much higher is this than last month? Which forwarder usually wins this lane? Draft an RFQ.*

Eventually recommends forwarders based on historical performance. Human approval always required.

---

# Module 3 — Stuffer Planner

## Purpose

Container planning before shipment execution. Determines how products fit into containers while coordinating with factories, and provides visibility into cargo readiness.

## Responsibilities

Container optimization · case planning · CBM calculations · cargo ready management · factory collaboration · quoting support · forecast loading dates

## Status — backend built, July 2026

Six `planner_` tables behind RLS. Internal sees every organization; a supplier sees only its own, and **sibling plants are grouped** — Junsun Thailand and Qingdao Junsun are one commercial relationship run out of two factories, so one login covers both, with a switcher between them. A rival factory sees nothing, including who works there.

Two model decisions worth carrying into other modules:

**CBM is dual-input.** Suppliers report either per-case or total, inconsistently. Both are stored as supplied, and the effective values are Postgres **generated columns** deriving one from the other against `quantity_available`. A generated column is writable by nobody, which makes the derivation impossible to bypass or disagree with — the app never performs that arithmetic.

**Cargo ready dates move, and the movement is the point.** `planner_po_line_events` is append-only, written by an AFTER UPDATE trigger, so a CSV upload and an inline grid edit produce identical history with no cooperation from the app. An upload that changes nothing writes nothing. This is `ARCHITECTURE.md` Part 5's event model in miniature, and currently the only place it exists.

Containers and allocations still resolve to in-memory local implementations; PO lines, suppliers and profiles come from the database.

## AI Usage

*Can these orders fit into two 40HC containers? What is preventing container completion? Which factory is delaying loading? Suggest better utilization.*

The AI assists. The planner remains responsible.

---

# Module 4 — Inbound Operations

## Purpose

The operational source of truth after shipments are created. Unlike the planning modules, Inbound represents what is *actually happening*.

**Everything the AI knows about shipment status originates here.**

## Responsibilities

Track shipments · containers · milestones · ETA updates · appointments · customer deliveries · maintain shipment history and event timelines

## Source of Truth

Inbound owns current shipment state, historical events, timelines, delivery progress, and operational milestones.

The AI never infers shipment state. It retrieves it from Inbound.

## Why Inbound Comes First For AI

Inbound has the **lowest decision complexity**. It answers factual questions:

*Where is container ABCD123456? Why did the ETA change? When is delivery expected? What happened yesterday? Which containers are delayed?*

The AI retrieves, explains, and summarizes. No business decisions are made. That makes Inbound the ideal first AI module — and the reason the AI roadmap in `AI.md` starts here rather than with the more visually impressive planning modules.

---

# Platform Services

## GeoBrain

Geocoding · routing · distance · location intelligence · geospatial cache management.

Next.js on Vercel. The only component permitted to call HERE Maps or OpenStreetMap directly.

See `ARCHITECTURE.md` Part 3.

---

# Guiding Principles

Every module owns its business domain.

Every module exposes reusable backend capabilities.

The dashboard consumes them. The AI consumes them. Background automation consumes them.

Business logic exists exactly once.

---

# Long-Term Vision

```
Planning  →  Execution  →  Intelligence
```

All operating on a single connected data model.

The AI is never the source of truth. It is the reasoning layer built upon the operational truth the platform creates.
