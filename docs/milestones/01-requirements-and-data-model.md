# TicketRush — System Design Learning Project

A BookMyShow / District-style **ticket flash sale** system, built from scratch to practise system design end to end: design, build, measure, deploy, explain.

- **Stack:** ASP.NET Core Web API (.NET 10), EF Core, PostgreSQL, React. k6 for load testing. Redis later — only if a measurement justifies it.
- **Shape:** one deployable backend + one database to start.
- **Scope:** the booking core (seats, holds, orders, payments). Not discovery, reviews, offers or dining.
- **Headline scenario:** 5,000 seats, ~50,000 people, sale opens at 10:00.

---

## Roadmap

| # | Milestone | In TicketRush | Status |
|---|---|---|---|
| 1 | Requirements & data model | Entities, keys, constraints, state machines, who wins a contested seat | **Design complete — review** |
| 2 | Vertical slice & architecture | Browse → hold → pay → ticket. Layers, dependency direction, DI lifetimes, why DbContext is scoped. Trace a click through middleware → auth → rules → EF → Postgres → response | |
| 3 | Read performance | Event listing + seat map over 100k+ seats. Filtering, stable sort, cursor vs offset pagination, projection, indexes. Measure plans and timings before/after | |
| 4 | Correctness & security | 10 people grab G-14 at once: conditional UPDATE vs optimistic concurrency vs row locks — measured. Idempotent payment. AuthN/AuthZ, JWT, BFF/CSRF | |
| 5 | Testing & operations | Unit + integration tests against Postgres, migrations, deployment, structured logs, correlation IDs, health checks. **First k6 load test** | |
| 6 | Async processing | Payment confirmation, ticket emails, hold-expiry worker, transactional outbox, retries with backoff, idempotent consumers, DLQ replay | |
| 7 | Integration & tenancy | Simulated payment gateway that slows/fails under load: timeouts, retries, circuit breaker, rate limits. Organisers/venues as tenants | |
| 8 | Architecture at scale | Virtual waiting room, rate limiting, Redis for hot reads, read model, Saga (late payment → refund). When partitioning/sharding is justified — with numbers | |

---

## 1. Requirements brief (v0)

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
4. Payment succeeds → seats **sold**, order confirmed, tickets issued.
5. She sees the booking and tickets under "My bookings".

### Business rules
- **R1** A seat is sold **at most once** per show — even with 50k concurrent users.
- **R2** A hold lasts 10 minutes; unpaid holds are released.
- **R3** Max 6 seats held per customer per event.
- **R4** A seat cannot be held while held by someone else or sold.
- **R5** The amount charged is the price at hold time, even if the organiser changes prices later.
- **R6** No double charge — double clicks and duplicate gateway callbacks are safe.
- **R7** A failed payment keeps the hold until expiry so the customer can retry.
- **R8** A payment that succeeds after its hold expired and the seat was resold is **automatically refunded**.

### Decisions on open questions
| Question | Decision | Why |
|---|---|---|
| Per-seat or group hold? | **One Hold covers a group of seats** (EventSeat.HoldId → Hold) | Priya's G-14 + G-15 expire and pay together; all-or-nothing is what customers expect |
| Late payment after resale? | **Refund automatically** (R8) | Money moved but nothing can be delivered → compensation. Mitigation (grace window while payment is in flight) — Milestone 6/8 |
| EventSeat rows up front? | **Yes — one row per seat per show, created when the event is set up** | 5,000 rows is tiny; gives one row per seat to lock/update atomically, and a simple seat-map query |
| Organiser cancels event? | **Out of scope for v0.** Later: Event → Cancelled, refund every confirmed order | Keep Milestone 1 focused on the sale itself |

---

## 2. Mental model used

| Bucket | Test | Entities |
|---|---|---|
| **Actors** | Who logs in or acts? | User (Customer / Organiser), Payment gateway (external) |
| **Catalog** | Described once; never "runs out" | Venue, Seat, Event |
| **Inventory** | Can run out; has a status | EventSeat — **availability lives here** |
| **Transactions** | Happened at a moment, done by someone | Hold, Order, OrderItem, Payment, Ticket |

Rules of thumb:
- **Status lives only on inventory and transactions**, never on catalog.
- **Verbs become tables:** hold → Hold, pay → Payment.
- **The "many" side holds the foreign key.**
- **Store each fact once.** No `AvailableSeats` counter on Event — it duplicates EventSeat and creates a hot row every booking must lock. Derive the count; cache only if measured.

---

## 3. Entity relationships

```
User ◀──────────── Hold ◀────────── EventSeat ──────▶ Seat ──────▶ Venue
 ▲                  ▲                   │  │                          ▲
 │                  │                   │  └────────▶ Event ──────────┘
 │                  │                   │               │
 ├──── Order ───────┘                   │               └──▶ User (organiser)
 │       ▲                              │
 │       ├──── OrderItem ───────────────┘
 │       │         ▲
 │       │         └──── Ticket
 │       └──── Payment
```
Arrows point from child (holds the FK) to parent.

| Relationship | Cardinality |
|---|---|
| Venue → Seats | 1 : many |
| Venue → Events | 1 : many |
| Event × Seat → EventSeat | many : many, resolved by EventSeat (one row per seat per show) |
| Hold → EventSeats | 1 : many (while held) |
| Order → OrderItems | 1 : many (one item per seat) |
| Order → Payments | 1 : many (retries after failure, R7); at most one Succeeded |
| OrderItem → Ticket | 1 : 0..1 (ticket issued on confirmation) |

---

## 4. Tables

Types are PostgreSQL. `timestamptz` ↔ C# `DateTimeOffset`. Money is `numeric(10,2)`, never float.

### User
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| Name | text | NOT NULL |
| Email | text | NOT NULL, **UNIQUE** |
| Phone | text | |
| Role | text | `Customer` \| `Organiser` — CHECK |
| CreatedAt | timestamptz | |

### Venue (catalog)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| Name | text | NOT NULL |
| City | text | NOT NULL |
| Address | text | |
| Capacity | int | CHECK (Capacity > 0) |

### Seat (catalog — the physical seat)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| VenueId | bigint | **FK → Venue** |
| Section | text | "Gold" |
| Row | text | "G" |
| Number | int | 14, CHECK (Number > 0) |

**UNIQUE (VenueId, Section, Row, Number)** — G-14 exists once per stadium.

### Event (catalog — the show)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| VenueId | bigint | **FK → Venue** |
| OrganiserId | bigint | **FK → User** |
| Title | text | NOT NULL |
| StartsAt | timestamptz | |
| EndsAt | timestamptz | CHECK (EndsAt > StartsAt) |
| SaleOpensAt | timestamptz | The flash-sale moment |
| Status | text | `Draft` \| `OnSale` \| `SoldOut` \| `Cancelled` \| `Completed` |

### EventSeat (inventory — seat × show)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| EventId | bigint | **FK → Event** |
| SeatId | bigint | **FK → Seat** |
| Price | numeric(10,2) | Current price for this seat at this show |
| Status | text | `Available` \| `Held` \| `Sold` |
| HoldId | bigint NULL | **FK → Hold**, set only while Held |
| Version | int | Optimistic concurrency (Milestone 4) |
| UpdatedAt | timestamptz | |

Constraints:
- **UNIQUE (EventId, SeatId)** — one row per seat per show.
- CHECK (Status IN ('Available','Held','Sold'))
- CHECK ((Status = 'Held') = (HoldId IS NOT NULL))
- Index (EventId, Status) — seat map and "seats left" queries.

### Hold (transaction — the 10-minute reservation)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| UserId | bigint | **FK → User** |
| EventId | bigint | **FK → Event** |
| Status | text | `Active` \| `Converted` \| `Expired` |
| ExpiresAt | timestamptz | CreatedAt + 10 min |
| CreatedAt | timestamptz | |

- Index (Status, ExpiresAt) — the expiry job finds `Active` holds past `ExpiresAt`.
- R3 (max 6 seats per customer per event) is checked inside the hold transaction; not a simple column constraint.

### Order (transaction — the purchase)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| UserId | bigint | **FK → User** |
| EventId | bigint | **FK → Event** |
| HoldId | bigint | **FK → Hold**, **UNIQUE** — one order per hold |
| Status | text | `Pending` \| `Confirmed` \| `Cancelled` \| `Refunded` |
| TotalAmount | numeric(10,2) | Sum of items + fees |
| Currency | char(3) | 'INR' |
| CreatedAt | timestamptz | |

### OrderItem (transaction — one seat in an order)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| OrderId | bigint | **FK → Order** |
| EventSeatId | bigint | **FK → EventSeat** |
| Price | numeric(10,2) | **Snapshot at hold time — R5** |

- **UNIQUE (OrderId, EventSeatId)**
- Why a separate table: price must be frozen per seat at hold time (R5), and each seat gets its own ticket.

### Payment (transaction — one attempt to pay)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| OrderId | bigint | **FK → Order** |
| Amount | numeric(10,2) | CHECK (Amount > 0) |
| Status | text | `Initiated` \| `Succeeded` \| `Failed` \| `Refunded` |
| IdempotencyKey | uuid | **UNIQUE** — a double-click sends the same key → one payment (R6) |
| GatewayPaymentId | text NULL | **UNIQUE** — a duplicate gateway callback finds the existing row (R6) |
| CreatedAt / UpdatedAt | timestamptz | |

- **Partial UNIQUE (OrderId) WHERE Status = 'Succeeded'** — an order can be paid successfully only once.

### Ticket (transaction — proof of purchase)
| Column | Type | Notes |
|---|---|---|
| Id | bigint | PK |
| OrderItemId | bigint | **FK → OrderItem**, **UNIQUE** — one ticket per seat bought |
| Code | text | **UNIQUE** — value encoded in the QR |
| IssuedAt | timestamptz | |

---

## 5. State machines

```
EventSeat:  Available ──hold──▶ Held ──payment succeeds──▶ Sold
                 ▲                │
                 └──hold expires──┘

Hold:       Active ──paid──▶ Converted
                 └──timer ran out──▶ Expired

Order:      Pending ──paid──▶ Confirmed
                 ├──hold expired──▶ Cancelled
                 └──paid too late──▶ Refunded

Payment:    Initiated ──gateway OK──▶ Succeeded ──(late)──▶ Refunded
                 └──declined / timeout──▶ Failed
```

- One business event moves several machines → **one DB transaction**.
- Sold → Available never happens automatically.

### Core invariant
**For one show, a seat has at most one active hold or sale.**
Enforced by: UNIQUE (EventId, SeatId) → exactly one row per seat per show; single `Status` column → that row is in exactly one state; conditional UPDATE (below) → only one request can move it out of `Available`.

---

## 6. Walk-throughs

### A. Priya's happy path (G-14 + G-15, Gold)
1. **Hold** — one DB transaction:
   - INSERT Hold (Priya, event 42, Active, ExpiresAt = now + 10 min)
   - `UPDATE EventSeat SET Status='Held', HoldId=@hold WHERE EventId=42 AND SeatId IN (G-14, G-15) AND Status='Available'` → must affect **2** rows, else ROLLBACK
   - INSERT Order (Pending, HoldId) + 2 OrderItems (Price = ₹4,000 each, frozen)
2. **Pay** — INSERT Payment (Initiated, IdempotencyKey) → redirect to gateway.
3. **Gateway callback: success** — one DB transaction:
   - Payment → Succeeded (lookup by GatewayPaymentId; already Succeeded → ignore duplicate)
   - Check Hold still Active → Hold → Converted
   - EventSeats → Sold, Order → Confirmed
   - INSERT 2 Tickets
4. Ticket email sent afterwards (async — Milestone 6).

### B. Hold expires before payment
- Expiry job (every few seconds): finds Holds with Status = Active AND ExpiresAt < now.
- One transaction per hold: Hold → Expired, its EventSeats → Available (HoldId = NULL), Order → Cancelled.
- G-14 shows as available again; Ravi can hold it.

### C. Late payment (R8)
- Callback arrives; Hold is already Expired and G-14 is Held by Ravi.
- Payment → Succeeded (the money did move), Order → Refunded, trigger gateway refund, notify Priya. EventSeat untouched.

### D. 10 people click G-14 in the same millisecond
All 10 run:
```sql
UPDATE event_seats
SET status = 'Held', hold_id = @myHold, version = version + 1, updated_at = now()
WHERE event_id = 42 AND seat_id = @g14 AND status = 'Available';
```
- Postgres locks the row for the first UPDATE that reaches it. The other 9 **wait** on that lock.
- The first commits: status is now `Held`.
- Each waiting UPDATE **re-checks the WHERE clause** against the committed row → `status = 'Available'` is false → **0 rows affected**.
- App sees 0 rows → rolls back its Hold → returns **409 Conflict: "Seat just taken"**.

Result: exactly 1 winner, decided by the database, not by an `if` in C#.
**Why not `if (seat.Status == Available) { seat.Status = Held; Save(); }`?** All 10 can read "Available" before any of them saves → 10 winners. That gap between check and write is the race condition.
Milestone 4 compares this against EF optimistic concurrency (`Version`) and `SELECT … FOR UPDATE`, with measurements.

---

## 7. Milestone 1 completion check
- [ ] Can explain, without notes, why Seat and EventSeat are separate tables
- [ ] Can name the unique key of Seat, EventSeat, Order(HoldId), Payment(IdempotencyKey, GatewayPaymentId), Ticket
- [ ] Can draw the four state machines from memory
- [ ] Can explain walk-through D and why the `if`-then-save version oversells
- [ ] Can explain why there is no `AvailableSeats` counter

**Realistic failure to test later (Milestone 4):** run the naive `if`-then-save version with 50 parallel requests for one seat and count how many "win".

### Interview question
> **"Design the booking flow for BookMyShow. How do you stop two users booking the same seat?"**

- **Model** — separate **catalog** (Venue, Seat, Event) from **inventory** (EventSeat: one row per seat per show) and **transactions** (Hold, Order, Payment, Ticket).
- **Invariant** — "one active hold or sale per seat per show": **UNIQUE (EventId, SeatId)** + a single **Status** column.
- **Concurrency** — **atomic conditional UPDATE** `… WHERE status = 'Available'`; the row lock makes 9 of 10 concurrent requests affect 0 rows → **409**. Not check-then-save.
- **Holds** — **10-minute TTL**, released by a background expiry job; seats grouped under one Hold.
- **Payments** — **idempotency key** for double clicks, **unique gateway id** for duplicate callbacks, **refund** when payment lands after the hold expired (compensation).
- **Example** — "10 users clicked G-14; one UPDATE matched, nine returned 0 rows and got 'seat just taken'."

---

## Review notes
- **2026-11-19:** First bucketing put Venue and Seat in Inventory and EventSeat in Transactions; fixed with the three-question test. Ticket moved from Inventory to Transactions.
- **2026-11-19:** Late-payment contradiction identified as Payment = Succeeded while the seat belongs to someone else → refund (R8). Invariant sharpened from "one person" to "one active hold or sale per EventSeat".
- **2026-11-20:** EventSeat draft lacked EventId, used booleans for status, linked straight to Customer. Fixed: EventId + SeatId FKs, single Status column, ownership via Hold.
- **2026-11-20:** Venue draft held show fields (start/end time) → moved to Event. `TotalSeats` decremented on booking rejected: duplicates truth + hot row.
- **2026-11-20:** Seat draft had FK pointing to EventSeat; corrected direction ("many side holds the FK") and added UNIQUE (VenueId, Section, Row, Number).
- **2026-11-20:** Hold, Order, OrderItem, Payment, Ticket provided by mentor — to be explained back.

---

## Learning approach
1. **Break it before fixing it** — build the naive version, make it fail, then fix.
2. **Decision log** — one short ADR per decision in `docs/decisions/`.
3. **Measure before claiming** — numbers and `EXPLAIN ANALYZE` in `docs/measurements.md`.
4. **Explain-it-back** — answer a short question per concept in my own words.
5. **Mock interview** every two milestones ("Design BookMyShow").
6. **Requirement twists** once something works.

### Load simulation — honest framing
- 50k **users** ≠ 50k requests/sec. k6 arrival-rate scenarios + think time.
- Track **p95 latency, error rate, oversold seats (must be 0)**.
- A Mac drives a few thousand VUs; rent a cloud VM for a true 50k run if needed.

## Tooling
- VS Code + C# Dev Kit · Docker Desktop (Postgres via `docker compose`) · DBeaver or VS Code PostgreSQL extension · REST Client `.http` files · ESLint + Prettier · k6 (Milestone 5)
- Run `dotnet ef`, `dotnet test`, `docker compose` from the terminal.
