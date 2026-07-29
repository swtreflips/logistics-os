                         React Apps

     Schedules        Rates        Planner        Inbound
          │             │             │             │
          └─────────────┴─────────────┴─────────────┘
                              │
                              ▼

                     Business Services API
                         (Railway)

              ┌──────────────┬──────────────┐
              │              │              │
        Shipment Service  Rate Service  Planner Service
        Schedule Service  Customer Service
              │
              ▼

           Supabase Database


                              ▲

                              │

                    GeoBrain Service
                     (Next.js/Vercel)

              ┌────────────────────────┐
              │                        │
              │ Geocoding              │
              │ Routing                │
              │ Distance               │
              │ Location Intelligence  │
              │ Cache Management       │
              │                        │
              └────────────────────────┘

---

# Service Boundaries

The platform runs **two backend deployables**. Both are backend services — both may talk to Supabase. Neither is a frontend extension.

## Business Services API

**Fastify on Railway** — `api.domain.com`

Owns logistics business logic: Shipment, Rate, Planner, Schedule, and Customer services. This is the service the React apps and, later, the AI tools call for anything domain-related.

Persistent Node process, which is what makes it the right home for background workers and LangGraph orchestration.

## GeoBrain Service

**Next.js on Vercel**

Owns everything geospatial: geocoding, routing, distance, location intelligence, and the cache that keeps those cheap.

Separate for good reasons:
- Geospatial results are highly cacheable and rarely change — a different scaling and caching profile from operational data
- HERE Maps calls cost money per request; one service with one cache means one place to control spend
- Request/response with no long-running work, which suits serverless well

**No business logic lives here.** GeoBrain answers "where is this and how far apart are these." It never knows what a shipment is.

---

# Why Next.js Here And Not Elsewhere

Next.js is used for GeoBrain **only**.

Frontends are Vite SPAs. The Business Services API is Fastify. GeoBrain is the single exception, justified by serverless caching economics — not a signal that Next.js is being adopted platform-wide.

---

# Open Questions

Not yet settled by this diagram. Tracked as D11 and D12 in `GAPS.md`.

**Who calls GeoBrain?**

- Business Services API only — one consumer, one cache, geospatial data enters the domain already enriched
- React apps directly — lower latency for map rendering, but two consumers to keep consistent
- Both, for different purposes

**Where does the GeoBrain cache live?**

The diagram shows GeoBrain writing to Supabase. If that's the shared operational database, the cache needs its own schema and must never be confused with business data. The alternative is a dedicated store (Vercel KV, or its own Postgres).

---

# The Rule That Doesn't Change

Two deployables, same layering discipline:

```
Interface  →  Service  →  Repository  →  Database
```

Splitting services across deployables is a hosting decision. It is not permission for a frontend to query the database directly, and it is not permission to duplicate business logic across the two.