# Glossary

One meaning per term, across every module, schema, service and eventual AI tool.

Drift here is expensive in a way that is hard to see coming. If Rates says `lane`, Planner says `route`, and Inbound says `shipment_leg` for the same thing, consolidation stops being an integration and becomes a translation project — and an assistant answering *"which lane was cheaper?"* has to guess which of three things the question means.

Where a term is **not yet settled**, that is stated. An honest open question is safer than a definition three apps quietly disagree with.

---

# Parties

## organization

The single party entity. One table, `organizations`, discriminated by `type`:

| `type` | Who | Sees |
|---|---|---|
| `internal` | Prime Time Packaging staff | everything |
| `forwarder` | freight forwarders quoting rates | their own quotes and lanes |
| `supplier` | factories producing goods | their own orders and cargo readiness |

There is **no separate `suppliers` or `customers` table**, and adding one would be a mistake — the isolation model, the group logic, and every RLS helper key on `organizations.id`.

`code` is a two-letter identifier used in container numbering. `active` gates visibility without deletion.

## organization group

Several organizations that are commercially one party. `organizations.group_id` points at `organization_groups`; `is_group_primary` marks the lead entity.

The motivating case: Junsun Packaging (Thailand) and Qingdao Junsun are **one relationship run out of two factories**. A user belonging to either sees both, and the UI offers a switch between them.

A group is *not* a corporate hierarchy and carries no ownership meaning. It answers exactly one question: which organizations should a person see as "theirs".

## supplier / factory

**The same thing.** `supplier` is the database term (`organizations.type = 'supplier'`); `factory` is the operational word people say, and what the Planner UI shows.

Deliberately not `customer` — these are parties the company **buys from**.

## forwarder

A freight forwarder quoting ocean rates. Untrusted-but-authenticated. The isolation requirement is absolute: Forwarder A must never see Forwarder B's quotes on a shared lane. That is competitive bid information, and it is the highest-stakes boundary in the platform.

## carrier

The ocean line operating the vessel — MSC, HPL, HMM, COSCO. **A carrier is not a forwarder.** A forwarder quotes a price; a carrier moves the box. Rates reference a carrier by name; Schedules keys on `carrier_code`.

## customer — ⚠️ NOT YET DEFINED (D9)

For an inbound importer this is genuinely ambiguous. It could be the end retailer, the DC receiving the goods, or the delivery recipient — and they are frequently different parties.

Nothing in the schema currently uses the word, which is the good news: **the decision is still free.** It will not stay free once Inbound starts, because Inbound is where deliveries acquire a recipient.

Define it before three modules define it differently. State explicitly what it is *not*.

---

# Places

## POL — Port of Loading

Origin ocean port. `pol` in Rates, `port_of_loading` in Schedules. **Same concept, two spellings** — a naming inconsistency to resolve in favour of `pol` when either is next touched.

## POD — Port of Discharge

Destination ocean port. Same split: `pod` in Rates, `port_of_discharge` in Schedules.

## FD — Final Destination

Inland delivery point, past the port. A rate to Dayton NJ has `pod` = a coastal port and `fd` = Dayton. The difference between POD and FD is the drayage leg, which is why `drayage_` rates are separate.

## last CY — last Container Yard

The final inland rail ramp or container yard on a routing. Carried by both Rates and Schedules (`last_cy`), and geocoded (`last_cy_geom`).

## origin / destination

Deliberately loose, used where a term must span ports and inland points. **Never use them where POL, POD, or FD is meant** — the precise term always wins in a schema column or a service signature.

---

# Commercial

## lane

**An origin–destination pair being priced.** In Rates, `rate_request_lanes` is one row per `(pol, fd, container_type)` within a batch.

A lane is a *pricing* concept, not a physical route. Two lanes may traverse identical water.

Reserve `lane` for pricing. Do not use it for a vessel routing — that is a **service** or a **routing**.

## rate

One forwarder's price for one lane, valid over a period. `rates` carries `rate_amount`, `currency`, `valid_from`, `valid_until`, `transit_days`, `free_days`, `carrier`.

**"Best rate" is a business rule, not a column** — it depends on validity window, container type, and free days. It belongs in one service, so that the screen and the assistant cannot disagree about it.

## RFQ batch

A round of rate requests sent to forwarders together — `rate_request_batches`, holding many lanes. The unit of "we went to market."

## submission

One forwarder's response to a batch — `rate_submissions`, containing many rates. Distinct from an individual `rate` so that "who responded, and when" is answerable without inspecting prices.

## OFQ

The committed container booking reference. A Planner container moves from `draft` to `committed` and receives an `ofq_reference`.

**Only internal commits.** A factory may build and fill a draft; it may not commit one.

---

# Cargo

## PO line

One item on one purchase order — `planner_po_lines`, the unit of planning.

Identified by **`(document_number, sku)`**. The internal export carries no line id, so any line number shown in the UI is derived for display and matches nothing. Enrichment files must join on this pair.

## document number

The purchase order identifier, `PO154984`. Called "Document Number" in every spreadsheet exchanged with suppliers, and the schema keeps that name deliberately.

## SKU / item

**The same thing.** `sku` in the schema, "Item" in supplier-facing spreadsheets and column headers.

## quantity vs quantity available

**`quantity`** is what was ordered. **`quantity_available`** is what remains to ship.

Planning always uses `quantity_available` — including CBM derivation. Using `quantity` there overstates every container.

## committed quantity

How much of a PO line is assigned to committed containers. A line is fully committed when `committed_quantity >= quantity_available`, at which point it leaves the open board.

## CBM

Cubic metres. **Dual-input by design**: suppliers report per-case or total, inconsistently and both are legitimate.

Both are stored as supplied. The effective values are Postgres **generated columns** deriving one from the other against `quantity_available`. Nobody can write a generated column, so the derivation cannot be bypassed or disagreed with — no application performs this arithmetic.

## cargo ready date — CRD

When a supplier will have goods ready to ship. **Supplier-owned; the only field they and internal both write.**

CRDs move, and *how far they move* is the point of tracking them. Never overwritten in place — `planner_po_line_events` appends every change, so slippage is a query rather than an anecdote.

## container

A physical shipping container being planned — `planner_containers`, `40HC` and similar, with `capacity_cbm` and a `destination`.

A container is **bound to one organization**. A container built under Junsun Thailand cannot hold Qingdao's lines, even though one person may see both.

Distinguish from a **container event** in Inbound, which is a milestone in a container's journey. Same word, different lifecycle stage — and a candidate for confusion once Inbound exists.

## allocation

The assignment of a quantity from a PO line into a container — `planner_allocations`. The many-to-many between cargo and boxes.

---

# Time

## ETD / ETA

**Estimated** time of departure / arrival. Forward-looking, revised often.

## ATD / ATA

**Actual** time of departure / arrival. Recorded once, never revised.

Never store an actual in an estimate column. The distinction is the entire basis of delay reporting: *"is it late"* is meaningless if the two are conflated.

## cutoff

The carrier's deadline for delivering cargo before a sailing. `cutoff_date` in Schedules.

## transit time

Days from departure to arrival. `transit_time_days` in Schedules, `transit_days` in Rates — **same concept, two spellings**, to converge when next touched.

---

# Movement

## schedule

A carrier's published sailing — `schedules`, keyed by `schedule_hash`, with `snapshot_date` because published schedules change and the changes matter.

## service / routing

A carrier's regular string of ports. **Not a lane** — a lane is priced, a service is operated.

## transshipment

Moving cargo between vessels at an intermediate port. `ts_ports` and `ts_vessels`. Direct sailings have none, and fewer transshipments generally means fewer failure points.

## drayage

Inland trucking between port and final destination — the POD-to-FD leg. Priced separately (`drayage_`) because it is a different market with different providers.

## shipment — ⚠️ NOT YET DEFINED

Will be owned by Inbound. Likely the execution-side unit — a booking in motion, carrying containers.

Do not use the word in Rates or Planner until Inbound defines it. It is the term most likely to be claimed early and wrongly.

---

# Open Terms

| Term | Blocked on |
|---|---|
| `customer` | D9 — end retailer, DC, or delivery recipient? |
| `shipment` | Inbound's data model |
| `booking` | overlaps OFQ; settle when Inbound exists |
| `leg` | not yet used; reserve for Inbound routing segments |

---

# Naming Inconsistencies To Resolve

Not urgent, not worth a migration. Fix each when the file is next touched.

| Concept | Rates | Schedules | Prefer |
|---|---|---|---|
| Port of loading | `pol` | `port_of_loading` | `pol` |
| Port of discharge | `pod` | `port_of_discharge` | `pod` |
| Transit time | `transit_days` | `transit_time_days` | `transit_days` |

---

# Rule

**A new domain term goes in this file the day it is invented** — before it appears in a second app, a service signature, or a column name.

The cost of adding a term here is a minute. The cost of two modules meaning different things by it is a translation layer that never fully goes away.
