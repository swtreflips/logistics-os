# design.md

# Design Principles

> **Build a logistics platform, not an AI application.**
>
> The AI is simply another consumer of the platform.

---

# The Fundamental Principle

Every business capability must exist independently of the user interface.

Never build features exclusively for the dashboard.

Instead, build reusable business services that can be consumed by:

- React Dashboard
- AI Agents (LangGraph)
- REST APIs
- Scheduled jobs
- Background workers
- Mobile applications
- Future integrations

The dashboard is only one interface.

The AI is another.

Both should use the exact same backend capabilities.

---

# The Wrong Architecture

```
React Page

↓

SQL Query

↓

Database
```

Problems

- Business logic duplicated
- Impossible for AI to reuse
- Hard to test
- Hard to maintain
- Tight coupling

---

# The Correct Architecture

```
                React Dashboard
                       │
                HTTP/API Request
                       │
               Container Controller
                       │
              Container Service
                       │
             Repository / Database
                       │
                  PostgreSQL
```

Later...

```
                LangGraph Agent
                       │
                    Tool
                       │
              Container Service
                       │
             Repository / Database
```

Notice

Neither React nor LangGraph knows how the business logic works.

They simply request capabilities.

---

# Everything Is A Capability

Never think

"I need a page."

Think

"I need a capability."

Examples

Instead of

```
Container Timeline Page
```

Think

```
getContainerTimeline(containerId)
```

Instead of

```
Shipment Search Screen
```

Think

```
searchShipments(filters)
```

Instead of

```
Delay Report
```

Think

```
findDelayedContainers()
```

The UI becomes a presentation layer over reusable capabilities.

---

# Example

Suppose Product requests:

"View Container Timeline"

The temptation:

```
React

↓

SELECT ...

↓

Display Timeline
```

Do NOT do this.

Instead build

```
ContainerService

getContainerTimeline(containerId)
```

Internally

```
Validate permissions

↓

Find container

↓

Retrieve events

↓

Sort chronologically

↓

Calculate current status

↓

Return DTO
```

Now this single capability can be used by

React

```
Timeline Page

↓

getContainerTimeline()
```

AI

```
User:

"What happened to container MSKU1234567?"

↓

Tool

↓

getContainerTimeline()

↓

LLM summarizes
```

Email Generator

```
Delay Notification

↓

getContainerTimeline()

↓

Generate explanation
```

Background Job

```
Daily Delay Report

↓

getContainerTimeline()
```

One implementation.

Many consumers.

---

# The AI Never Talks To The Database

Wrong

```
LangGraph

↓

SQL

↓

Database
```

Correct

```
LangGraph

↓

Tool

↓

Business Service

↓

Repository

↓

Database
```

The AI should only know what tools exist.

It should never know SQL.

---

# Build Services, Not Endpoints

Avoid writing business logic directly inside API routes.

Bad

```
GET /containers/:id

↓

50 lines of SQL

↓

Return JSON
```

Good

```
GET /containers/:id

↓

ContainerService.getContainer()

↓

Return JSON
```

The API layer should remain thin.

---

# Controllers Should Coordinate

Controllers should

- Validate request
- Authenticate user
- Call service
- Return response

Controllers should NOT

- Build SQL
- Calculate ETA
- Determine shipment status
- Compute inventory
- Execute business rules

---

# Services Own Business Logic

Every business rule belongs inside services.

Examples

ContainerService

ShipmentService

InventoryService

PurchaseOrderService

CustomerService

NotificationService

EmailService

TaskService

ScheduleService

AppointmentService

---

# Services May Span Deployables

The services above live in the Business Services API. Geospatial capability lives in a separate GeoBrain Service.

That split is a **hosting decision**, made for caching and cost reasons. It changes nothing about layering:

```
Interface  →  Service  →  Repository  →  Database
```

Both deployables follow it.

Splitting across deployables is never permission to:

- Let a frontend query the database directly
- Duplicate business logic so two services can each "own" it
- Put business rules into a platform service

When deciding where a new capability belongs, ask whether it is **logistics domain logic** or a **general-purpose capability**. Domain logic goes in Business Services. General-purpose capability may earn its own service — but only when caching, cost, or scaling profile actually justifies the second deployable. Two services are a cost; pay it deliberately.

See `SERVICES.md`.

---

# Repositories Own Data Access

Repositories only retrieve or persist data.

No business decisions.

No AI logic.

No formatting.

Example

```
ContainerRepository

findById()

findEvents()

findByBooking()

findByCustomer()
```

---

# Every Feature Should Produce A Tool

When implementing a feature ask

"What tool will this become?"

Example

Today

```
Feature

View Purchase Order
```

Tomorrow

```
Tool

getPurchaseOrder(poId)
```

---

Today

```
Feature

Search Inventory
```

Tomorrow

```
Tool

findInventory(itemNumber)
```

---

Today

```
Feature

Customer Shipment Summary
```

Tomorrow

```
Tool

getCustomerInboundSummary(customerId)
```

Design for tomorrow while building today.

---

# Service First Development

Preferred order

1. Database Schema

↓

2. Repository

↓

3. Business Service

↓

4. API Route

↓

5. React Page

Never the opposite.

---

# Business Logic Should Exist Once

Bad

```
Dashboard computes ETA

API computes ETA

AI computes ETA
```

Three implementations.

Eventually inconsistent.

Good

```
ShipmentService

calculateETA()
```

Everyone calls it.

---

# LangGraph Integration

LangGraph should orchestrate work.

It should NOT contain business logic.

Example

User

"Where is item 501234?"

Graph

```
Find Item Tool

↓

Get Container Tool

↓

Get Timeline Tool

↓

Summarize
```

Every node calls an existing backend service.

---

# AI Tools Mirror Business Services

If a service exists

```
ShipmentService

getShipmentTimeline()
```

There should eventually be

```
Tool

getShipmentTimeline()
```

Do not write AI-specific implementations.

Wrap existing services.

---

# Human And AI Are Equal Clients

The system should not distinguish between

```
Dashboard

↓

ShipmentService
```

and

```
LangGraph

↓

ShipmentService
```

Only authentication and authorization differ.

Business logic remains identical.

---

# Authorization Always Happens In Services

Never trust

- React
- LangGraph
- API callers

Services must verify

- User identity
- Organization
- Permissions
- Access rights

Every time.

---

# Long-Term Outcome

The platform should eventually support

React Dashboard

↓

Shipment Services

↓

Database

AND

LangGraph AI Assistant

↓

Shipment Services

↓

Database

AND

Background Automation

↓

Shipment Services

↓

Database

All business intelligence lives inside the platform.

Not inside React.

Not inside LangGraph.

Not inside prompts.

---

# Guiding Question

Every time a new feature is proposed, ask:

> **If we removed the React UI tomorrow, would this capability still exist?**

If the answer is **no**, the feature has been implemented in the wrong layer.

The platform should own capabilities.

Interfaces should simply expose them.