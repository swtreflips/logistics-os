# Architecture

How the Logistics OS is built: layering, service boundaries, technology, data, and security.

---

# Part 1 — The Fundamental Principle

Every business capability must exist independently of the user interface.

Never build features exclusively for the dashboard. Build reusable business services consumable by:

- React dashboards
- AI agents (LangGraph)
- REST APIs
- Scheduled jobs and background workers
- Future mobile apps and integrations

The dashboard is one interface. The AI is another. Both use the exact same backend capabilities. Only authentication differs.

## Human First, In Sequence And In Priority

The human applications are the foundation. The AI is a layer on top of them.

This is not just ordering — it is a standard. Each app must stand entirely on its own for a human operator with no AI present. Build the dashboard as though the assistant will never exist, and build the *backend* as though it certainly will.

In practice that means the frontend contains no AI affordances and no gaps waiting to be filled by one, while the layering below it is designed from day one so that a tool can call any capability the UI can.

The apps are currently in active development, human-only. That is the correct phase. See `ROADMAP.md` for what "laying the groundwork" concretely requires while that continues.

## Everything Is A Capability

Never think *"I need a page."* Think *"I need a capability."*

| Instead of | Think |
|---|---|
| Container Timeline Page | `getContainerTimeline(containerId)` |
| Shipment Search Screen | `searchShipments(filters)` |
| Delay Report | `findDelayedContainers()` |

The UI becomes a presentation layer over reusable capabilities.

## Every Feature Produces A Tool

When implementing a feature, ask: *"What tool will this become?"*

| Today — Feature | Tomorrow — Tool |
|---|---|
| View Purchase Order | `getPurchaseOrder(poId)` |
| Search Inventory | `findInventory(itemNumber)` |
| Customer Shipment Summary | `getCustomerInboundSummary(customerId)` |

Design for tomorrow while building today.

---

# Part 2 — Layering

```
Interface  →  Service  →  Repository  →  Database
```

Every deployable follows this. No exceptions.

## Wrong

```
React Page  →  SQL  →  Database
```

Business logic duplicated, impossible for the AI to reuse, hard to test, tightly coupled.

## Correct

```
React Dashboard              LangGraph Agent
      │                            │
 HTTP Request                    Tool
      │                            │
      └──────► Container Service ◄─┘
                     │
              Repository
                     │
                PostgreSQL
```

Neither React nor LangGraph knows how the business logic works. They request capabilities.

## Responsibilities

**Interfaces** (React, LangGraph tools, API clients) render, collect input, manage UI state, navigate. Nothing more.

**Controllers / Routes** validate the request, authenticate the user, call a service, return the response. They never build SQL, calculate ETA, determine status, or execute business rules.

**Services** own every business rule. Permissions, calculations, ETA logic, status derivation, orchestration.

**Repositories** retrieve and persist. No business decisions, no formatting, no AI logic.

```
ContainerRepository
  findById()
  findEvents()
  findByBooking()
  findByCustomer()
```

## Service-First Development

```
1. Database Schema
2. Repository
3. Business Service
4. API Route
5. React Page
```

Never the reverse.

## Worked Example

Product asks for "View Container Timeline." The temptation is React → SQL → display.

Build instead `ContainerService.getContainerTimeline(containerId)`:

```
Validate permissions → Find container → Retrieve events
→ Sort chronologically → Calculate current status → Return DTO
```

One implementation. Four consumers:

| Consumer | Use |
|---|---|
| React | Timeline page |
| AI | "What happened to MSKU1234567?" |
| Email generator | Delay notification body |
| Background job | Daily delay report |

---

# Part 3 — The Two Deployables

The platform runs two backend services. Both are backend services; both may talk to Supabase. Neither is a frontend extension.

```
                    React Apps

    Schedules     Rates     Planner     Inbound
         │          │          │           │
         └──────────┴──────────┴───────────┘
                         │
                         ▼

     Business Services API         GeoBrain Service
          (Fastify/Railway)         (Next.js/Vercel)

    ┌────────────────────────┐   ┌──────────────────────┐
    │ Shipment Service       │   │ Geocoding            │
    │ Container Service      │──▶│ Routing              │
    │ Rate Service           │   │ Distance             │
    │ Schedule Service       │   │ Location Intelligence│
    │ Planner Service        │   │ Cache Management     │
    │ Customer Service       │   └──────────────────────┘
    │ AI Tool Service        │              │
    └────────────────────────┘              │
                  │                         │
                  ▼                         ▼

                       Supabase
               PostgreSQL + Auth + Storage
```

## Business Services API

**Fastify on Railway** — `api.domain.com`

Owns all logistics business logic. New services go here by default. A persistent Node process, which is what makes it the right home for background workers and LangGraph orchestration.

Structure:

```
logistics-api/src/
├── routes/
├── services/
├── repositories/
├── ai/
├── middleware/
├── integrations/
└── utils/
```

Core services: `ShipmentService`, `ContainerService`, `InventoryService`, `PurchaseOrderService`, `CustomerService`, `RateService`, `ScheduleService`, `PlannerService`, `NotificationService`, `EmailService`, `TaskService`, `AppointmentService`.

## GeoBrain Service

**Next.js on Vercel**

Owns geocoding, routing, distance, location intelligence, and the cache that keeps those cheap.

Separate for real reasons:

- Geospatial results are highly cacheable and rarely change — a different scaling profile from operational data
- HERE Maps bills per request; one service with one cache means one place to control spend
- Request/response with no long-running work, which suits serverless

**No business logic lives here.** GeoBrain answers *where is this* and *how far apart are these*. It never knows what a shipment is.

## Why Next.js Here And Nowhere Else

Next.js is used for GeoBrain **only**. Frontends are Vite SPAs. The Business Services API is Fastify. This exception is justified by serverless caching economics and must not spread.

## The Rule That Doesn't Change

Splitting services across deployables is a **hosting decision**. It is never permission to:

- Let a frontend query the database directly
- Duplicate business logic so two services can each "own" it
- Put business rules into a platform service

When placing a new capability, ask whether it is **logistics domain logic** or a **general-purpose capability**. Domain logic goes in Business Services. A general-purpose capability may earn its own deployable — but only when caching, cost, or scaling profile justifies it. Two services are a cost; pay it deliberately.

---

# Part 4 — Technology Stack

## Frontend

React · Vite · TypeScript

Frontends are Vite SPAs, one per module, one subdomain each. Never Next.js.

## Backend

Node.js · Fastify · TypeScript — Business Services API

Next.js — GeoBrain Service only

## Database

Supabase · PostgreSQL · Row Level Security

## Authentication

Supabase Auth. Centralized identity across all applications.

## Hosting

| Platform | Hosts |
|---|---|
| **Vercel** | Frontend applications (subdomain per module), GeoBrain Service |
| **Railway** | Business Services API, background workers |

Railway is chosen for the API because it provides always-on Node processes, GitHub-based deploys, Docker support, and environment management. The backend is a permanent platform service, not a frontend extension.

## Storage

Supabase Storage

## Maps

HERE Maps · OpenStreetMap (where applicable)

**Accessed only through the GeoBrain Service.** No frontend, service, or worker calls a maps provider directly.

## Email

Microsoft Graph API — customer communications, document processing, notifications.

## AI

Anthropic Claude · LangGraph · LangChain (tool definitions and integrations where appropriate)

## Observability

Sentry · OpenTelemetry · structured logging. Add when there is production traffic worth observing.

## Not Yet Settled

Do not treat these as decided. See the Decisions Log in `ROADMAP.md`.

- Data grid — rates-app uses MUI X DataGrid, stufferPlanner uses ag-Grid (D6)
- Component system — MUI vs. Radix (D7)
- Server state — React Query intended, not yet adopted
- Forms — React Hook Form intended, not yet adopted

## Deployment Model

```
GitHub
   │
   ├── React Projects ──────▶ Vercel
   │
   ├── GeoBrain Service ────▶ Vercel ──────┐
   │                                       │
   └── Business Services API ▶ Railway ────┤
                                           ▼
                                       Supabase
```

## External Integrations

**Ocean carriers** — schedule and shipment data collection via Python scrapers and APIs where available.

**Microsoft Graph** — email automation.

**HERE Maps** — geospatial, through GeoBrain only.

---

# Part 5 — Data

## Normalize

Everything references IDs. Avoid duplicated text.

Core entities: Customers · Factories · Suppliers · Items · Purchase Orders · Item Allocations · Bookings · Containers · Container Events · Shipments · Shipment Events · Carriers · Vessels · Ports · Carrier Schedules · Rates · Documents · Appointments · Emails · Notifications · Tasks

## Event Driven

**Never overwrite shipment status. Store events.**

```
Container MSKU1234567

Loaded → Departed Shanghai → Delayed → ETA Updated
→ Arrived Busan → Departed Busan → Arrived LA
→ Customs Released → Out for Delivery → Delivered
```

Current status is always derived from the event history. Historical events are never deleted.

## Events And Projections

Deriving status on every read will not meet the performance target, and it makes *"show me all delayed containers"* expensive and awkward to express in SQL.

The resolution:

> **Derive state from events. Materialize it for reads. Never let anything write the projection directly.**

A current-state projection table is maintained on event insert. It is duplicated state, deliberately, and it is always reconstructible by replaying events. This is the one sanctioned exception to "business state exists once."

## Audit Trail

Every consequential action records:

- Who
- When
- Old value / new value
- Reason
- Whether AI was involved
- Prompt ID, if applicable
- Which human approved

Audit columns (`created_by`, `updated_by`) belong on every business table from creation. Retrofitting them loses all history prior to the change.

---

# Part 6 — Security

## Authorization Always Happens In Services

Never trust React, LangGraph, or an API caller. Services verify user identity, organization, permissions, and access rights — every time.

## The AI Has No Special Access

If a user cannot see a shipment in the UI, the AI cannot see it for them. Every tool executes under the caller's identity. RLS is never bypassed to make an agent convenient.

## Unresolved

The authoritative layer is **not yet decided** — see D2 and D3 in `ROADMAP.md`. Currently RLS does all authorization because the browser talks to Postgres directly. Once the API sits in between, one of these must be chosen:

- **User JWT forwarded to Supabase** — RLS stays authoritative, services add rules on top
- **Service role key** — services become authoritative, RLS bypassed server-side

Background jobs and AI agents have no user session, so a service identity concept is required either way.

Decide before writing the first Fastify route.

---

# Part 7 — Engineering Standards

## Code

TypeScript everywhere. Strict typing. No `any`.

Prefer composition. Small services. Small files. Dependency injection where practical. Consistent naming.

## Errors

Never expose stack traces. Return meaningful errors. Log everything. Recover gracefully.

Every service defines its not-found and permission-denied shapes explicitly. A permission denial must not reveal whether the record exists.

## Testing

Unit · integration · tool · agent workflow · permission · regression.

## Performance

| Target | Goal |
|---|---|
| Dashboard API | <200ms where possible |
| AI conversation | 5–10s |
| Database | Indexed foreign keys, no N+1, batch operations, cache where appropriate |

Independent lookups inside a service run in parallel, not sequentially.

---

# Long-Term Outcome

```
React Dashboard  ─┐
LangGraph Agent  ─┼─▶  Business Services  ─▶  Database
Background Jobs  ─┘
```

All business intelligence lives inside the platform. Not in React. Not in LangGraph. Not in prompts.
