# CLAUDE.md

# Inbound Logistics Dashboard & AI Operations Assistant

## Vision

Build an enterprise-grade inbound logistics platform that provides complete visibility into shipments, containers, purchase orders, inventory, and customers.

The application must first function as an exceptional human-operated dashboard. Every feature should then be designed so an AI agent can access the exact same business capabilities through tools and APIs.

The AI is **not the source of truth.**

The database and business services are.

The AI's responsibility is to:

- Retrieve information
- Understand context
- Summarize events
- Answer questions
- Draft communications
- Execute approved business actions

Never allow the AI to invent shipment information.

---

# Core Philosophy

Human Dashboard First.

AI Second.

Everything the dashboard can do should eventually become available through a secure backend service.

The frontend should never contain business logic.

Business logic belongs exclusively in the backend.

The AI should never access the database directly.

Instead it should call backend tools.

Architecture should be designed from day one with this separation.

---

# Tech Stack

See `STACK.md` for the full architecture. This is the summary.

Frontend

- React
- Vite
- TypeScript

No Next.js. Frontends are Vite SPAs.

Backend

- Node.js
- Fastify
- TypeScript

A standalone `logistics-api` service — not API routes inside a frontend app.

Database

- Supabase
- PostgreSQL
- Row Level Security

Authentication

- Supabase Auth

Hosting

- Vercel — frontend applications, one subdomain per module
- Railway — the Logistics API and background workers

Storage

- Supabase Storage

Maps

- HERE Maps
- OpenStreetMap (where applicable)

Email

- Microsoft Graph API

AI

- Anthropic Claude
- LangGraph
- LangChain (tool definitions and integrations where appropriate)

## Not Yet Settled

Do not treat these as decided. See the Decisions Log in `GAPS.md`.

- Data grid — rates-app uses MUI X DataGrid, stufferPlanner uses ag-Grid (D6)
- Component system — MUI vs. Radix (D7)
- Server state — React Query intended, not yet adopted in either app
- Forms — React Hook Form intended, not yet adopted in either app

Observability

- Sentry
- OpenTelemetry
- Structured logging

---

# Long-Term Vision

Eventually the system becomes an AI Logistics Operations Assistant capable of:

- Answering shipment questions
- Understanding shipment history
- Tracking containers
- Finding inventory
- Finding customers
- Summarizing delays
- Drafting customer emails
- Creating operational tasks
- Explaining ETA changes
- Comparing historical performance
- Detecting operational risks
- Recommending actions

---

# Development Phases

## Phase 1

Inbound Dashboard

Human only.

Focus:

- Fast
- Accurate
- Easy to use

No AI assumptions in UI.

Instead:

Design backend correctly.

---

## Phase 2

Backend Service Layer

Every dashboard operation should become a reusable service.

Good:

ContainerService.getContainer()

Bad:

Database queries inside React components.

Every service should be usable by:

- UI
- AI
- Future API
- Mobile app

---

## Phase 3

AI Read-only Assistant

Users can ask:

Where is container ABCD123456?

Which containers are delayed?

Show timeline.

What happened yesterday?

Why did ETA change?

No write operations.

---

## Phase 4

AI Action Assistant

Allow AI to:

Draft emails

Draft updates

Create follow-up tasks

Generate customer summaries

Still require human approval.

---

## Phase 5

AI Operations Agent

Proactive monitoring.

Example:

Container delayed 4 days.

↓

Find affected customers.

↓

Estimate impact.

↓

Generate summaries.

↓

Notify planners.

↓

Prepare customer communications.

---

# Architecture Principles

## Single Source of Truth

Shipment information lives only in PostgreSQL.

Never duplicate business state inside prompts.

---

## Event Driven

Never overwrite shipment status.

Store events.

Example:

Container

MSKU1234567

Events

Loaded

Departed Shanghai

Delayed

ETA Updated

Arrived Busan

Departed Busan

Arrived LA

Customs Released

Out for Delivery

Delivered

The current status should always be derived from the latest event.

Historical events are never deleted.

---

## Thin Frontend

Frontend responsibilities:

- Render UI
- Validate user input
- Display data

Nothing more.

---

## Rich Backend

Backend responsibilities:

- Business rules
- Calculations
- ETA logic
- Permissions
- Tool execution
- Logging

---

# AI Philosophy

The LLM should never answer using memory alone.

Every logistics question should retrieve data first.

The AI's job is reasoning.

Not storage.

---

# LangGraph Design

The AI layer will use LangGraph as the orchestration engine.

Each business capability becomes a Tool.

Example tools:

findContainer()

findItem()

findCustomer()

findPurchaseOrder()

shipmentTimeline()

currentContainerStatus()

estimatedArrival()

containersForCustomer()

inventoryForItem()

findDelayedContainers()

findAffectedCustomers()

draftDelayEmail()

draftArrivalEmail()

createFollowUpTask()

scheduleReminder()

createInternalNote()

Every tool wraps backend services.

Never direct SQL.

---

# Example Agent Flow

User

"Where is item 500123?"

↓

Locate Item Tool

↓

Container Tool

↓

Shipment Timeline Tool

↓

Summarize

↓

Return answer

---

User

"Email Sweetgreen explaining the delay."

↓

Customer Tool

↓

Shipment Tool

↓

Draft Email Tool

↓

Return draft

↓

User approves

↓

Send Email Tool

---

# Human Approval

AI may NEVER automatically:

Send email

Modify shipment

Delete records

Change ETA

Close tasks

Generate invoices

Release inventory

Anything with business consequences requires approval.

---

# Database Philosophy

Normalize data.

Examples

Containers

Container Events

Purchase Orders

Customers

Items

Item Allocations

Bookings

Ports

Vessels

Carrier Schedules

Documents

Appointments

Emails

Notifications

Tasks

Everything references IDs.

Avoid duplicated text.

---

# Audit Trail

Every important action must be recorded.

Who

When

Old value

New value

Reason

Whether AI was involved

Prompt ID (if applicable)

User approval

---

# AI Memory

Do not store logistics state in memory.

Conversation memory should only remember:

Current conversation

Temporary references

User preferences

Shipment data must always come from tools.

---

# Security

AI respects user permissions.

If the user cannot access a shipment through the UI,

the AI cannot access it.

Never bypass RLS.

Every tool executes under the authenticated user's identity.

---

# Future Tool Categories

Shipment Tools

Inventory Tools

Customer Tools

Purchase Order Tools

Booking Tools

Schedule Tools

Email Tools

Reporting Tools

Analytics Tools

Forecasting Tools

Document Tools

Task Tools

Notification Tools

---

# Coding Standards

TypeScript everywhere.

Strict typing.

No "any".

Prefer composition.

Small services.

Small files.

Dependency injection where practical.

Reusable utility functions.

Consistent naming.

---

# Error Handling

Never expose stack traces.

Return meaningful errors.

Log everything.

Recover gracefully.

---

# Testing

Unit tests

Integration tests

Tool tests

Agent workflow tests

Permission tests

Regression tests

---

# Performance Goals

Dashboard

<200ms API response where possible

AI

Most conversations under 5-10 seconds

Database

Indexed foreign keys

Avoid N+1 queries

Batch operations

Caching where appropriate

---

# Success Metric

A logistics coordinator should eventually be able to ask:

"What happened to Sweetgreen's shipment?"

"Which customers will be delayed next week?"

"Draft updates for everyone affected."

"What containers contain item 500123?"

"What changed since yesterday?"

…and receive accurate answers backed entirely by live operational data.

The AI is an intelligent operations assistant built on top of a reliable logistics platform—not a replacement for it.