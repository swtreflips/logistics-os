# AI_EXAMPLE.md

# AI Shipment Assistant Example

## Use Case

A user wants to understand the current status and history of an inbound shipment.

The user should not need to manually navigate:

- shipment tables
- container records
- ETA history
- cargo details
- customer impact reports

The AI assistant should provide a business-level explanation by using existing logistics capabilities.

---

# User Perspective

## User Interface

The user opens the AI assistant inside the Inbound Dashboard.

Example:


Inbound Shipment Assistant

User:

"Give me a summary of INBSHIP001"

AI:

INBSHIP001 is currently in transit from Shanghai
to Dallas.

The shipment was originally expected to arrive
on August 5, but the ETA has moved twice and is
currently expected on August 12.

The shipment is currently delayed by 7 days.

The container contains:

500 cases of Kraft Shopping Bag Large
700 cases of Kraft Shopping Bag Medium

Both ETA revisions were recorded as vessel schedule
changes: the first on July 25, the second on July 29.

Would you like me to draft a customer update email?


---

# What the User Sees

The user does NOT see:

- Database queries
- Table relationships
- API calls
- Service names
- Internal logic

The user only interacts with:

- Natural language
- Business explanations
- Recommended actions

---

# AI Reasoning Flow

User:


"Give me a summary of INBSHIP001"


↓

AI determines that it needs shipment context.

↓

AI calls:


getInboundShipmentSummary("INBSHIP001")


↓

Inbound Service provides structured information.

↓

AI generates a human-readable response.

---

# High-Level Architecture

                User

                 │

                 ▼

         AI Assistant UI

                 │

                 ▼

          LangGraph Agent

                 │

                 ▼

         Available Tools

                 │

                 ▼

      Logistics Business Services

                 │

                 ▼

          Supabase Database

---

# Tool Layer

The AI does not directly access the database.

Instead, it has access to business capabilities.

Example tools:


getInboundShipmentSummary()

getContainerTimeline()

getCargoContents()

getCustomerImpact()

draftDelayEmail()


---

# Example Tool Definition

## getInboundShipmentSummary

Purpose:

Return all relevant operational information required to explain an inbound shipment.

Input:

```json
{
  "shipment_id": "INBSHIP001"
}
```

Output:

```json
{
  "shipment_id": "INBSHIP001",

  "status": "In Transit",

  "carrier": "MSC",

  "container": "MSKU1234567",

  "eta": {
    "current_eta": "2026-08-12",
    "original_eta": "2026-08-05",
    "delay_days": 7
  },

  "eta_history": [
    {
      "date": "2026-07-22",
      "eta": "2026-08-05",
      "reason": null
    },
    {
      "date": "2026-07-25",
      "eta": "2026-08-09",
      "reason": "VESSEL_SCHEDULE_CHANGE"
    },
    {
      "date": "2026-07-29",
      "eta": "2026-08-12",
      "reason": "VESSEL_SCHEDULE_CHANGE"
    }
  ],

  "cargo": [
    {
      "item": "Kraft Shopping Bag Large",
      "cases": 500
    },
    {
      "item": "Kraft Shopping Bag Medium",
      "cases": 700
    }
  ],

  "origin": "Shanghai",

  "destination": "Dallas"
}
```

## On the `reason` field

The assistant may only state *why* an ETA moved if the platform recorded a reason.

Where `reason` is `null`, the correct response is to say the reason was not recorded — never to infer one from timing, carrier, or lane.

An assistant that explains a delay the database cannot explain is the failure mode this entire architecture exists to prevent.

---

# Backend Service Flow

The AI tool calls the Inbound Service.

Example:

```
LangGraph

     │

     ▼

getInboundShipmentSummary()

     │

     ▼

Inbound Service API

     │

     ├── ShipmentService
     │
     ├── TimelineService
     │
     ├── CargoService
     │
     └── ETAService

     │

     ▼

Supabase
```

---

# Service Implementation Example

The service should orchestrate business capabilities.

The four lookups do not depend on each other, so they run in parallel.

```ts
async function getInboundShipmentSummary(shipmentId) {

    const [shipment, timeline, cargo, etaHistory] =
        await Promise.all([
            shipmentService.getShipment(shipmentId),
            timelineService.getTimeline(shipmentId),
            cargoService.getCargo(shipmentId),
            etaService.getHistory(shipmentId)
        ])

    return {
        shipment,
        timeline,
        cargo,
        etaHistory
    }
}
```

---

# Database Interaction

The services interact with the database.

Example:

```
Inbound Service

        │

        ├── shipments table
        |
        ├── containers table
        |
        ├── shipment_events table
        |
        ├── eta_history table
        |
        └── container_items table


        │

        ▼

     Supabase
```

The AI does not know these tables exist.

---

# AI Response Generation

The AI receives the tool output defined above. Dates stay in ISO form — formatting is the assistant's job, not the service's.

```json
{
  "status": "In Transit",
  "eta": {
    "original_eta": "2026-08-05",
    "current_eta": "2026-08-12",
    "delay_days": 7
  },
  "cargo": [
    { "item": "Kraft Shopping Bag Large", "cases": 500 },
    { "item": "Kraft Shopping Bag Medium", "cases": 700 }
  ],
  "destination": "Dallas"
}
```

The AI converts this into:

```
INBSHIP001 is currently delayed by 7 days.

The original ETA was August 5 and has moved to
August 12 after two schedule adjustments.

The container contains 1,200 cases across two
product lines, bound for Dallas.
```

Every claim above traces to a field in the payload: the delay to `delay_days`, the dates to `eta`, the case count to the sum of `cargo[].cases`, the destination to `destination`.

Nothing else may be said.

---

# Future Extension: Customer Communication

The user asks:

"Draft an email to the customer explaining the delay"

The AI workflow becomes:

```
getInboundShipmentSummary()

          ↓

getCustomerInformation()

          ↓

calculateImpact()

          ↓

draftDelayEmail()

          ↓

Human Approval

          ↓

Send Email
```

The summary is re-fetched at draft time. A shipment's ETA can move between the question and the email, and the draft must reflect what is true when it is written — never a payload cached earlier in the conversation.

---

# Core Design Principle

The AI is not the logistics system.

The AI is an intelligent interface over the logistics system.

The intelligence comes from:

- accurate operational data
- well-defined business services
- reusable capabilities
- structured workflows

The AI provides:

- interpretation
- explanation
- communication
- orchestration

The platform provides:

- truth
- calculations
- business rules
- operational knowledge

---

# The Test For Any Response

Every sentence the assistant produces must trace to a field returned by a tool.

If it cannot, the assistant either omits it or says the platform does not record it.

A fluent answer that outruns its data is worse than no answer — it is wrong in a way the user cannot detect.