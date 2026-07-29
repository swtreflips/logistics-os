# STACK.md

# Logistics OS Technology Stack

## Overview

The Logistics OS is designed as a modular operational platform.

The architecture separates:

- User interfaces
- Business logic
- Data storage
- AI orchestration
- Background processing

Each layer has a clear responsibility.

The platform is built to support:

- Human dashboards
- APIs
- AI assistants
- Automation workflows
- Future integrations

---

# High-Level Architecture

```
                    Users

                      │

                      ▼

              React Applications

        ┌─────────┬─────────┬─────────┐
        │         │         │         │
   Schedules   Rates    Planner   Inbound

                      │

                      ▼

        Business Services API          GeoBrain Service
             (Railway)                  (Next.js/Vercel)

        ┌─────────────────────────┐   ┌──────────────────────┐
        │                         │   │                      │
        │ Shipment Services       │   │ Geocoding            │
        │ Container Services      │──▶│ Routing              │
        │ Rate Services           │   │ Distance             │
        │ Schedule Services       │   │ Location Intelligence│
        │ Planner Services        │   │ Cache Management     │
        │ Customer Services       │   │                      │
        │ AI Tool Services        │   └──────────────────────┘
        │                         │              │
        └─────────────────────────┘              │
                      │                          │
                      ▼                          ▼

                          Supabase

                  PostgreSQL + Auth + Storage

                      │

                      ▼

          AI Layer (LangGraph)

                      │

                      ▼

              Anthropic Claude
```

---

# Frontend Layer

## Framework

React + Vite

## Purpose

Frontend applications provide human interfaces into the Logistics OS.

The frontend should:

- Display information
- Collect user input
- Manage UI state
- Handle navigation
- Provide visualization

The frontend should NOT:

- Contain business rules
- Calculate logistics logic
- Directly implement complex database queries
- Duplicate backend functionality

---

# Frontend Applications

## Schedules App

Purpose:

Provide visibility into carrier schedules and routing options.

Responsibilities:

- Search schedules
- Compare carriers
- Display transit times
- Show transshipment options

---

## Rates App

Purpose:

Provide a centralized freight rate management system.

Responsibilities:

- Receive forwarder rates
- Compare quotes
- Track historical pricing
- Support quoting workflows

---

## Stuffer Planner

Purpose:

Support container loading and cargo planning.

Responsibilities:

- Manage cargo readiness
- Calculate CBM
- Plan container configurations
- Coordinate factory updates

---

## Inbound Dashboard

Purpose:

Provide operational visibility after shipments are created.

Responsibilities:

- Track containers
- Display shipment status
- Show milestone history
- Monitor delays
- Provide operational truth

This module becomes the foundation for the first AI assistant phase.

---

# Backend Layer

The platform runs two backend deployables. See `SERVICES.md` for boundaries.

## Business Services API

Runtime: Node.js

Framework: Fastify

Hosting: Railway

Owns all logistics business logic. New services go here by default.

## GeoBrain Service

Framework: Next.js

Hosting: Vercel

Owns geocoding, routing, distance, location intelligence, and geospatial caching. No business logic. It never knows what a shipment is.

Next.js is used here and nowhere else on the platform.

---

# Business Services API

The Business Services API is the central business layer.

All applications interact with business capabilities through the API.

The API is responsible for:

- Business rules
- Data aggregation
- Permissions
- Validation
- AI tool exposure
- Integration logic

---

# Backend Architecture

```
logistics-api

src/

├── routes/

├── services/

├── repositories/

├── ai/

├── middleware/

├── integrations/

└── utils/
```

---

# Services Layer

Business logic lives inside services.

Examples:

## ShipmentService

Capabilities:

```
getShipment()

getShipmentTimeline()

findDelayedShipments()

calculateETA()

```

---

## ContainerService

Capabilities:

```
getContainer()

getContainerContents()

getContainerEvents()

```

---

## CustomerService

Capabilities:

```
getCustomer()

getCustomerShipments()

getCustomerImpact()

```

---

# Repository Layer

Repositories handle database interaction.

Repositories should:

- Query data
- Insert records
- Update records

Repositories should NOT:

- Contain business rules
- Calculate logistics decisions
- Format user responses

---

# Hosting Layer

## Frontend Hosting

Platform:

Vercel

Purpose:

Host React applications.

Examples:

```
schedules.domain.com

rates.domain.com

planner.domain.com

inbound.domain.com
```

---

## Backend Hosting

Platform:

Railway

Purpose:

Host the Business Services API.

Example:

```
api.domain.com
```

Responsibilities:

- Run Node.js services
- Handle API requests
- Expose business capabilities
- Serve AI workflows

---

# Why Railway

Railway provides:

- Always available backend services
- GitHub-based deployments
- Docker support
- Environment management
- Scalable infrastructure

The backend is treated as a permanent platform service, not a frontend extension.

---

# Database Layer

## Platform

Supabase

## Database

PostgreSQL

---

# Database Responsibilities

Supabase manages:

- Operational data
- Authentication
- Storage
- Permissions
- Relational models

Core entities:

```
Customers

Factories

Items

Purchase Orders

Containers

Shipments

Shipment Events

Carriers

Vessels

Ports

Rates

Schedules

Tasks

Documents
```

---

# Authentication

Platform:

Supabase Auth

All applications use centralized identity.

Permissions must be enforced through:

- Supabase policies
- Backend validation
- Service-level authorization

---

# AI Layer

## Framework

LangGraph

## Model Provider

Anthropic Claude

---

# AI Architecture

The AI is not the source of truth.

The AI reasons over business capabilities exposed by the Business Services API.

```
User

↓

LangGraph

↓

AI Tools

↓

Business Services API

↓

Supabase

↓

Response

↓

Claude Explanation
```

---

# AI Tools

Examples:

```
getContainerTimeline()

findShipmentByItem()

findDelayedContainers()

getCustomerImpact()

draftDelayEmail()

```

Tools should call backend services.

The AI should never directly query the database.

---

# AI Development Phases

## Phase 1

Inbound Assistant

Capabilities:

- Shipment lookup
- Timeline explanations
- ETA explanations
- Delay summaries

Read-only.

---

## Phase 2

Communication Assistant

Capabilities:

- Draft customer emails
- Draft internal updates
- Generate reports

Human approval required.

---

## Phase 3

Operational Assistant

Capabilities:

- Monitor exceptions
- Create tasks
- Identify affected customers
- Recommend actions

---

## Phase 4

Planning Intelligence

Capabilities:

- Recommend schedules
- Compare rates
- Suggest container plans

Human decisions remain final.

---

# Background Processing

Future components:

- Shipment monitoring
- Scheduled updates
- Carrier tracking
- Email automation
- AI agents

Recommended future infrastructure:

Railway workers

or

Cloud Run jobs

---

# External Integrations

## Ocean Carriers

Purpose:

Schedule and shipment data collection.

Technology:

Python scrapers

APIs where available

---

## Microsoft Graph

Purpose:

Email automation.

Uses:

- Customer communications
- Document processing
- Notifications

---

## HERE Maps

Purpose:

Geospatial calculations.

Uses:

- Routing
- Distance calculations
- Location intelligence

**Accessed only through the GeoBrain Service.** No frontend, service, or worker calls a maps provider directly. HERE bills per request, so a single cached entry point is both an architectural and a cost decision.

---

# Deployment Model

```
GitHub

    │

    ├── React Projects ──────▶ Vercel

    │

    ├── GeoBrain Service ────▶ Vercel

    │                             │
    │                             ▼
    │                          Supabase
    │
    └── Business Services API ▶ Railway

                                  │
                                  ▼
                               Supabase
```

---

# Engineering Principles

## Business Logic Once

Never implement the same logic in:

- React
- AI prompts
- Scripts
- Reports

Business logic belongs in services.

---

## APIs Before Interfaces

Build capabilities first.

Example:

Not:

"Build container timeline page."

Instead:

"Build getContainerTimeline()."

Then create interfaces around it.

---

## AI Is A Consumer

The AI is another client of the platform.

It is not the platform.

---

# Long-Term Vision

The Logistics OS becomes a connected operational system where:

Planning

↓

Execution

↓

Analytics

↓

AI Intelligence

all operate on the same business model.

The competitive advantage is not the AI model.

The competitive advantage is the operational data model and the business capabilities built around it.