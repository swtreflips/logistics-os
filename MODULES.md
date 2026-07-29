# MODULES.md

# Logistics OS Modules

## Vision

The Logistics OS is composed of independent but interconnected modules.

Each module owns a specific portion of the logistics lifecycle while exposing reusable business capabilities through backend services.

Modules are **not isolated applications**.

Together they form a single operational platform.

Each module contributes information to a shared operational model that eventually powers the AI Operations Assistant.

---

# Platform Lifecycle

```
Planning
    │
Schedules
    │
Rates
    │
Container Planning
    │
Shipment Execution
    │
Inbound Operations
    │
AI Operations Assistant
```

Planning creates shipments.

Execution tracks shipments.

AI understands shipments.

---

# Business Modules vs. Platform Services

The four modules below own **business domains**. Each has a lifecycle stage, a user, and its own operational meaning.

Alongside them sit **platform services** — shared capabilities with no business domain of their own. They are consumed by modules but never own logistics state.

Current platform services:

**GeoBrain** — geocoding, routing, distance, location intelligence, geospatial caching. See `SERVICES.md`.

The test: if a capability would still make sense in a company that wasn't in logistics, it is a platform service, not a module. GeoBrain passes that test. Inbound does not.

Platform services do not appear in the module lifecycle and do not get their own AI roadmap phase. They make modules possible.

---

# Module 1

# Schedules

## Purpose

Help logistics coordinators determine the best ocean freight routing before booking.

The module aggregates carrier schedules into one searchable interface.

Instead of checking multiple carrier websites manually, users compare all options from one place.

---

## Primary Responsibilities

Collect schedules from carriers.

Compare transit times.

Compare transshipment ports.

Identify direct sailings.

Track schedule changes.

Present routing alternatives.

---

## Data Sources

Carrier websites

Carrier APIs

Python scraping services

---

## Future AI Usage

Answer questions such as

- What's the fastest routing?
- Which carrier has the fewest transshipments?
- Which services arrive before August 10?
- Compare ONE and Maersk for this lane.

The AI recommends.

Humans decide.

---

# Module 2

# Rates

## Purpose

Create a collaborative rate management platform between the company and freight forwarders.

Replace spreadsheets and email exchanges with structured data.

Provide a central source of truth for quoted freight rates.

---

## Primary Responsibilities

Receive quotes.

Store historical rates.

Compare forwarders.

Track validity periods.

Manage lane pricing.

Maintain quoting history.

---

## Future AI Usage

Answer questions such as

- Who quoted the best rate?
- How much higher is this than last month?
- Which forwarder usually wins this lane?
- Draft RFQs.

Eventually recommend forwarders based on historical performance.

Human approval always required.

---

# Module 3

# Stuffer Planner

## Purpose

Assist with container planning before shipment execution.

Determine how products fit into containers while coordinating with factories.

Provide visibility into cargo readiness.

---

## Primary Responsibilities

Container optimization.

Case planning.

CBM calculations.

Cargo Ready management.

Factory collaboration.

Quoting support.

Forecast loading dates.

---

## Future AI Usage

Questions like

- Can these orders fit into two 40HC containers?
- What is preventing container completion?
- Which factory is delaying loading?
- Suggest better container utilization.

The AI assists planners.

The planner remains responsible.

---

# Module 4

# Inbound Operations

## Purpose

Serve as the operational source of truth after shipments have been created.

This module represents what is actually happening in the supply chain.

Unlike planning modules, Inbound focuses on execution.

Everything the AI knows about shipment status originates here.

---

## Primary Responsibilities

Track shipments.

Track containers.

Track milestones.

Track ETA updates.

Maintain shipment history.

Maintain event timelines.

Track appointments.

Track customer deliveries.

Maintain operational visibility.

---

## Why This Module Comes First For AI

Inbound has the lowest decision complexity.

It primarily answers factual questions.

Examples

Where is container ABCD123456?

Why did ETA change?

When is delivery expected?

What happened yesterday?

Which containers are delayed?

The AI retrieves.

The AI explains.

The AI summarizes.

No business decisions are made.

This makes Inbound the ideal first AI module.

---

## Source of Truth

Inbound owns

Current shipment state.

Historical shipment events.

Shipment timelines.

Delivery progress.

Operational milestones.

The AI should never infer shipment state.

It retrieves it from Inbound.

---

# AI Operations Assistant

The AI is **not** a separate business module.

It is an intelligence layer built on top of every module.

Its capabilities expand as each module exposes reusable backend services.

---

# AI Roadmap

## Phase 1

Inbound Assistant

Read-only.

Capabilities

Answer shipment questions.

Explain delays.

Summarize timelines.

Locate containers.

Locate inventory.

Explain ETA changes.

Generate operational summaries.

No write operations.

No automation.

Human asks.

AI answers.

---

## Phase 2

Communication Assistant

The AI begins producing work.

Draft delay emails.

Draft arrival notices.

Draft customer updates.

Draft internal summaries.

Everything requires approval before sending.

---

## Phase 3

Operational Assistant

The AI begins coordinating work.

Find affected customers.

Identify delayed purchase orders.

Create follow-up tasks.

Generate daily reports.

Highlight operational risks.

Still human-approved.

---

## Phase 4

Planning Assistant

AI expands into planning modules.

Schedules

Rates

Stuffer Planner

Capabilities

Recommend sailings.

Compare forwarders.

Suggest container configurations.

Estimate costs.

Recommend bookings.

Provide planning insights.

Humans remain decision makers.

---

## Phase 5

Cross-Module Intelligence

The AI reasons across the entire Logistics OS.

Example

"What happened to Sweetgreen's shipment?"

AI workflow

Locate shipment.

↓

Retrieve schedule.

↓

Retrieve booking.

↓

Retrieve container.

↓

Retrieve cargo.

↓

Retrieve latest events.

↓

Retrieve appointments.

↓

Retrieve customer.

↓

Generate explanation.

The AI now understands the complete shipment lifecycle.

---

# Guiding Principles

Every module owns its own business domain.

Every module exposes reusable backend capabilities.

The dashboard consumes those capabilities.

The AI consumes those capabilities.

Background automation consumes those capabilities.

Business logic exists exactly once.

---

# Long-Term Vision

The Logistics OS becomes a complete operational platform where

Planning

↓

Execution

↓

Intelligence

operate from a single connected data model.

The AI is never the source of truth.

The AI is the reasoning layer built upon the operational truth created by the platform.