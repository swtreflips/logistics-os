# AI Operations Assistant

The AI is not a business module. It is an intelligence layer over every module, and its capabilities expand as each module exposes reusable services.

---

# Philosophy

The LLM never answers from memory. Every logistics question retrieves data first.

**The AI's job is reasoning. Not storage.**

| The AI provides | The platform provides |
|---|---|
| Interpretation | Truth |
| Explanation | Calculations |
| Communication | Business rules |
| Orchestration | Operational knowledge |

The AI is an intelligent interface over the logistics system. It is not the logistics system.

---

# The Grounding Rule

> **Every sentence the assistant produces must trace to a field a tool returned.**
>
> If it cannot, the assistant says the platform does not record it.

This is the single most important rule in this document, and the easiest to violate without noticing — because a model that infers a plausible delay reason produces text that reads *better* than one that admits ignorance.

A fluent answer that outruns its data is worse than no answer. It is wrong in a way the user cannot detect, and it will be forwarded to a customer.

## In Practice

If the payload has no `reason` field, the assistant may not explain *why* an ETA moved. Not from timing. Not from the carrier. Not from the lane. Not from what usually happens.

If `reason` is `null`, the correct response is **"the reason was not recorded."**

## Never Store State In Memory

Conversation memory holds only the current conversation, temporary references, and user preferences.

Shipment data always comes from tools. On a follow-up turn, re-fetch — an ETA can move between the question and the answer.

---

# Tool Design

## Tools Wrap Services

If a service exists, there should eventually be a tool that calls it. Never write AI-specific implementations of business logic.

```
LangGraph  →  Tool  →  Business Service  →  Repository  →  Database
```

The AI knows what tools exist. It never knows SQL, and it never learns that tables exist.

## Prefer Coarse-Grained Tools

Return everything needed to answer a question in one call, rather than making the model chain four fine-grained lookups.

`getInboundShipmentSummary()` beats `getShipment()` → `getTimeline()` → `getCargo()` → `getETAHistory()`.

Fewer round trips, lower latency, and far fewer chances for the model to compose a sequence incorrectly. The fine-grained version looks more elegant and costs weeks in tool-chaining bugs.

## Every Tool Defines Its Failure Shapes

Not found, permission denied, and empty result are all explicit contract elements. Without them the model fills silence with invention.

A permission denial must not reveal whether the record exists.

---

# Tool Catalog

**Shipment** — `findContainer` · `currentContainerStatus` · `shipmentTimeline` · `estimatedArrival` · `findDelayedContainers` · `getInboundShipmentSummary`

**Inventory** — `findItem` · `inventoryForItem` · `getCargoContents`

**Customer** — `findCustomer` · `containersForCustomer` · `findAffectedCustomers` · `getCustomerImpact` · `getCustomerInboundSummary`

**Purchase Order** — `findPurchaseOrder` · `getPurchaseOrder`

**Booking & Schedule** — booking lookup · carrier schedule comparison · routing alternatives

**Rates** — quote comparison · historical pricing · forwarder performance

**Communication** — `draftDelayEmail` · `draftArrivalEmail` · `createInternalNote`

**Tasks** — `createFollowUpTask` · `scheduleReminder`

**Reporting** — operational summaries · analytics · forecasting

---

# Human Approval

The AI may **never** automatically:

Send email · modify a shipment · delete records · change an ETA · close tasks · generate invoices · release inventory

Anything with business consequences requires human approval. The AI drafts; a person commits.

---

# Security

The AI respects user permissions absolutely.

If a user cannot access a shipment through the UI, the AI cannot access it for them. Every tool executes under the caller's identity. RLS is never bypassed to make an agent convenient.

Agents and background jobs have no user session, so they require a service identity with explicitly scoped permissions — see D3 in `ROADMAP.md`.

---

# AI Phases

These are **AI phases**, distinct from the platform stages in `ROADMAP.md`. AI Phase A1 cannot begin until Platform Stage P2 (service layer extracted) is complete.

## A1 — Inbound Assistant

**Read-only. No writes, no automation.**

Answer shipment questions · explain delays · summarize timelines · locate containers · locate inventory · explain ETA changes · generate operational summaries.

Human asks. AI answers.

## A2 — Communication Assistant

The AI begins producing work.

Draft delay emails · arrival notices · customer updates · internal summaries.

Everything requires approval before sending.

## A3 — Operational Assistant

The AI begins coordinating work.

Find affected customers · identify delayed purchase orders · create follow-up tasks · generate daily reports · highlight operational risks.

Still human-approved.

## A4 — Planning Intelligence

The AI expands into Schedules, Rates, and Stuffer Planner.

Recommend sailings · compare forwarders · suggest container configurations · estimate costs.

Humans remain decision makers.

## A5 — Cross-Module Intelligence

The AI reasons across the entire platform.

> *"What happened to Sweetgreen's shipment?"*

```
Locate shipment → Retrieve schedule → Retrieve booking
→ Retrieve container → Retrieve cargo → Retrieve latest events
→ Retrieve appointments → Retrieve customer → Generate explanation
```

Eventually proactive: a container is delayed four days → find affected customers → estimate impact → generate summaries → notify planners → prepare communications. Every outbound action still gated on a human.

---

# Worked Example

## The Question

User opens the assistant inside the Inbound Dashboard:

> **"Give me a summary of INBSHIP001"**

## The Flow

```
User
 │
 ▼
AI Assistant UI
 │
 ▼
LangGraph Agent
 │
 ▼
getInboundShipmentSummary("INBSHIP001")
 │
 ▼
Inbound Service
 ├── ShipmentService
 ├── TimelineService
 ├── CargoService
 └── ETAService
 │
 ▼
Supabase
```

The service orchestrates. The four lookups are independent, so they run in parallel:

```ts
async function getInboundShipmentSummary(shipmentId) {
  const [shipment, timeline, cargo, etaHistory] = await Promise.all([
    shipmentService.getShipment(shipmentId),
    timelineService.getTimeline(shipmentId),
    cargoService.getCargo(shipmentId),
    etaService.getHistory(shipmentId)
  ])
  return { shipment, timeline, cargo, etaHistory }
}
```

## The Payload

```json
{
  "shipment_id": "INBSHIP001",
  "status": "In Transit",
  "carrier": "MSC",
  "container": "MSKU1234567",
  "origin": "Shanghai",
  "destination": "Dallas",
  "eta": {
    "original_eta": "2026-08-05",
    "current_eta": "2026-08-12",
    "delay_days": 7
  },
  "eta_history": [
    { "date": "2026-07-22", "eta": "2026-08-05", "reason": null },
    { "date": "2026-07-25", "eta": "2026-08-09", "reason": "VESSEL_SCHEDULE_CHANGE" },
    { "date": "2026-07-29", "eta": "2026-08-12", "reason": "VESSEL_SCHEDULE_CHANGE" }
  ],
  "cargo": [
    { "item": "Kraft Shopping Bag Large", "cases": 500 },
    { "item": "Kraft Shopping Bag Medium", "cases": 700 }
  ]
}
```

Dates stay ISO. Formatting is the assistant's job, not the service's.

## The Response

```
INBSHIP001 is currently in transit from Shanghai to Dallas.

The shipment was originally expected on August 5, but the ETA
has moved twice and is currently expected on August 12 — a
delay of 7 days.

Both revisions were recorded as vessel schedule changes: the
first on July 25, the second on July 29.

The container contains 1,200 cases across two product lines:
500 cases of Kraft Shopping Bag Large and 700 of Medium.

Would you like me to draft a customer update email?
```

## Why This Response Is Legitimate

Every claim traces to a field:

| Claim | Source |
|---|---|
| In transit, Shanghai → Dallas | `status`, `origin`, `destination` |
| August 5 → August 12, moved twice | `eta`, `eta_history` |
| 7 days | `delay_days` |
| Vessel schedule changes, July 25 and 29 | `eta_history[].reason` |
| 1,200 cases, two lines | sum of `cargo[].cases` |

**Nothing else may be said.** Not which customers are affected, not what it means for the warehouse, not why the vessel was rescheduled — none of that is in the payload.

## What The User Never Sees

Database queries · table relationships · API calls · service names · internal logic.

Only natural language, business explanations, and recommended actions.

---

# Extension: Customer Communication

> **"Draft an email to the customer explaining the delay"**

```
getInboundShipmentSummary()   ← re-fetched, not reused
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

The summary is re-fetched at draft time. An ETA can move between the question and the email, and the draft must reflect what is true when written.

---

# Known Gaps In This Design

Not yet solved. Tracked in `ROADMAP.md`.

**Entity resolution.** The example assumes the user types `INBSHIP001`. Real users say *"the Sweetgreen shipment,"* *"the one arriving Tuesday,"* or give a container or PO number. There is no `findShipment(query)` step. This is the most likely reason a working demo fails on first contact with a real coordinator — nearly every real conversation starts with resolution, not lookup.

**Identity.** The flow shows no auth context anywhere. Blocked on D2 and D3.

**Audit.** Nothing in this flow records that the AI read or drafted anything, though the audit requirements in `ARCHITECTURE.md` demand it.

**Error contract.** Not-found and permission-denied shapes are unspecified.
