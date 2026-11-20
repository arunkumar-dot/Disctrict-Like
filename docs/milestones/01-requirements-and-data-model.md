# TicketRush — System Design Learning Project

A BookMyShow / District-style **ticket flash sale** system, built from scratch to practise system design end to end: design, build, measure, deploy, explain.

- **Stack:** ASP.NET Core Web API (.NET 10), EF Core, PostgreSQL, React. k6 for load testing. Redis later — only if a measurement justifies it.
- **Shape:** one deployable backend + one database to start.
- **Scope:** the booking core (seats, holds, orders, payments). Not discovery, reviews, offers or dining.
- **Headline scenario:** 5,000 seats, ~50,000 people, sale opens at 10:00.

> Why not Purchase Approval? It teaches workflows and authorization well, but has no real contention — 50k people never act at once. TicketRush keeps every roadmap concept and adds concurrency and load that can be simulated honestly.

---

## Roadmap

| # | Milestone | In TicketRush | Status |
|---|---|---|---|
| 1 | Requirements & data model | Event, Seat, Hold, Order, Payment; seat state machine; who wins a contested seat | **In progress** |
| 2 | Vertical slice & architecture | Browse → hold → pay → ticket. Layers, dependency direction, DI lifetimes, why DbContext is scoped. Trace a click through middleware → auth → rules → EF → Postgres → response | |
| 3 | Read performance | Event listing + seat map over 100k+ seats. Server-side filtering, stable sort, cursor vs offset pagination, projection, indexes. Measure SQL plans and timings before/after | |
| 4 | Correctness & security | 10 people grab G-14 at once: optimistic concurrency vs row locks vs unique constraint vs atomic conditional UPDATE — measured. Idempotent payment. AuthN/AuthZ, JWT, BFF/CSRF | |
| 5 | Testing & operations | Unit + integration tests against Postgres, migrations, deployment, structured logs, correlation IDs, health checks. **First k6 load test** | |
| 6 | Async processing | Payment confirmation, ticket emails, hold expiry worker, transactional outbox, retries with backoff, idempotent consumers, DLQ replay | |
| 7 | Integration & tenancy | Simulated payment gateway that slows/fails under load: timeouts, retries, circuit breaker, rate limits. Organisers/venues as tenants | |
| 8 | Architecture at scale | Virtual waiting room, rate limiting, Redis for hot reads, read model, Saga (payment fails → release seats). When would partitioning/sharding be justified — with numbers | |

---

## Milestone 1 — Requirements brief (v0)

### Scenario
A popular concert at a Bengaluru venue goes on sale Saturday 10:00. 5,000 seats in sections — **Platinum ₹8,000, Gold ₹4,000, Silver ₹1,500**. ~50,000 people waiting.

### Actors
- **Customer** — browses events, picks seats, pays, gets a ticket.
- **Organiser** — creates an event, sets seat layout and prices, opens the sale.
- **System** — background jobs: expires holds, confirms payments, sends tickets.
- **Payment gateway** — external; can be slow, fail, or call back twice.

### Happy path
1. Priya opens the event and sees the seat map with live availability.
2. She selects Gold G-14 and G-15. They are **held** for her for **10 minutes**.
3. She pays ₹8,000 + fees via the gateway.
4. Payment succeeds → seats **sold**, order confirmed, ticket issued.
5. She sees the booking and ticket under "My bookings".

### Business rules (starter set)
- **R1** A seat is sold **at most once** — must hold even with 50k concurrent users.
- **R2** A hold lasts 10 minutes; unpaid holds are released.
- **R3** Max 6 seats held per customer per event.
- **R4** A seat cannot be held while held by someone else or sold.
- **R5** The amount charged is the price at hold time, even if the organiser changes prices later.
- **R6** No double charge — double clicks and duplicate gateway callbacks are safe.
- **R7** A failed payment keeps the hold until expiry so the customer can retry.

### Open questions (decide and defend)
- [ ] Per-seat holds, or one hold covering a group of seats?
- [ ] Payment succeeds 2 seconds after the hold expired and someone else took the seat — what happens?
- [ ] 5,000 `Seat` rows per event up front, or rows only for held/sold seats?
- [ ] Can the organiser cancel an event? What happens to paid orders?

---

## Milestone 1 — Design sketch (to fill in)

### Entities
Starting suggestion: `Venue`, `Event`, `Seat` / `EventSeat`, `Hold`, `Order`, `Payment`, `User`. Add, merge, or drop with reasons.

| Entity | Key fields & types | PK / FKs | Constraints |
|---|---|---|---|
| | | | |

### Relationships
- Cardinalities:
- Is a seat's physical position in the venue the same thing as its availability for one event?

### State transitions
- **Seat:**
- **Order:**
- **Payment:**

### Walk-throughs
1. Priya's happy path:
2. Hold expires before payment:
3. **10 people click G-14 in the same millisecond — which constraint or mechanism guarantees exactly one wins?**

### Review notes
_(mentor review goes here)_

---

## Learning approach
1. **Break it before fixing it** — build the naive version, make it fail (oversell, slow query), then fix.
2. **Decision log** — one short ADR per decision in `docs/decisions/` (context, options, choice, trade-off, how verified).
3. **Measure before claiming** — record numbers and `EXPLAIN ANALYZE` output in `docs/measurements.md`.
4. **Explain-it-back** — answer a short question per concept in my own words; refine.
5. **Mock interview** every two milestones ("Design BookMyShow").
6. **Requirement twists** once something works, to test design boundaries.

### Load simulation — honest framing
- 50k **users** ≠ 50k requests/sec. Model with k6 arrival-rate scenarios + think time.
- Track **p95 latency, error rate, and oversold seats (must be 0)**.
- A Mac drives a few thousand VUs — enough to break a naive design and prove a good one. Rent a cloud VM for a true 50k run if needed.

---

## Tooling
- VS Code + **C# Dev Kit**
- **Docker Desktop** for PostgreSQL (`docker compose`)
- DB client: VS Code PostgreSQL extension or DBeaver
- REST Client / Thunder Client with `.http` files committed
- ESLint + Prettier (React)
- `brew install k6` (from Milestone 5)
- Habit: run `dotnet ef`, `dotnet test`, `docker compose` from the terminal, not IDE buttons.
