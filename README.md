# Intelligent Railway Ticket Booking & Allocation Management System
### Master Architectural Roadmap

---

## PART 1 — ARCHITECTURAL OVERVIEW

### 1.1 System Vision

A full-stack, enterprise-grade railway reservation platform that behaves like a real production system (search, book, cancel, waitlist, RAC, refunds, live seat maps, TTE/conductor tools, station-master operations, admin analytics) while its backend is deliberately engineered as a **showcase of advanced DBMS and distributed-systems concepts**. Every "advanced concept" is not bolted on — it is the natural implementation choice for a real railway problem (zones are naturally distributed, seat locking is naturally row-level, fare rules are naturally semi-structured, availability lookups are naturally cache-heavy).

### 1.2 Recommended Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite + TailwindCSS + Redux Toolkit + React Query | Fast agent-buildable SPA, strong typing, good for real-time seat maps |
| Realtime | Socket.IO / WebSocket gateway | Live seat-map + waitlist-position updates |
| API Layer | Node.js + Express + TypeScript | Client-server tier, REST + WS, thin orchestration over PL/SQL |
| Primary RDBMS | **Oracle Database (23ai Free or 21c XE)** | True PL/SQL, Object Types, row-level locking (`SELECT ... FOR UPDATE`), partitioning, DB Links |
| Distributed Simulation | 3 Oracle **Pluggable Databases / schemas** = "Zones" (Northern, Western, Southern Railway Zone) linked via **DB Links** | Parallel & Distributed DB concept, data sharded geographically like real Indian Railways zones |
| NoSQL #1 | **Redis** | Seat-availability cache, session store, distributed lock for hot seats |
| NoSQL #2 | **MongoDB** | Semi-structured data: dynamic fare rules, train amenities, audit/event logs, notification templates |
| Auth | JWT (access + refresh) + bcrypt + RBAC | Passenger / TTE / StationMaster / ZoneAdmin / SuperAdmin |
| Containerization | Docker Compose | Orchestrates Oracle, Redis, Mongo, API, Frontend as one client-server unit |
| API Docs | OpenAPI/Swagger | Contract-first, agent-verifiable |
| Testing | Jest + Supertest (API), Playwright (E2E) | Verifiable phase-by-phase correctness |

### 1.3 EER Model — Core Specialization/Generalization Hierarchies

- **PERSON (generalization)** → `PASSENGER`, `EMPLOYEE`
  - `EMPLOYEE` (generalization) → `TTE_CONDUCTOR`, `STATION_MASTER`, `ZONE_ADMIN`
- **TRAIN** (generalization, disjoint) → `EXPRESS_TRAIN`, `SUPERFAST_TRAIN`, `PASSENGER_TRAIN`, `FREIGHT_TRAIN`
- **TICKET** (generalization, overlapping→state machine) → `CONFIRMED_TICKET`, `RAC_TICKET`, `WAITLISTED_TICKET`
- **PAYMENT** (generalization) → `UPI_PAYMENT`, `CARD_PAYMENT`, `WALLET_PAYMENT`
- Core aggregation entities: `ZONE`, `STATION`, `ROUTE`, `TRAIN_SCHEDULE`, `COACH`, `SEAT`, `BOOKING`, `WAITLIST_QUEUE`, `FARE_RULE`

### 1.4 ADBMS Concept → Implementation Mapping

| Concept | Where it lives in the system |
|---|---|
| EER Modeling | Oracle schema: supertype/subtype tables with discriminator + FK-as-PK pattern |
| PL/SQL (procedures, triggers, locking) | `booking_pkg`, `waitlist_pkg`, `fare_pkg`; `SELECT FOR UPDATE` on `SEAT`; `AFTER INSERT`/`INSTEAD OF` triggers on ticket cancellation auto-promoting RAC→Confirmed |
| Object-Oriented DB | Oracle `CREATE TYPE ... AS OBJECT` for `ADDRESS_T`, `JOURNEY_SEGMENT_T` (nested table/VARRAY), type methods (`MEMBER FUNCTION`) |
| Parallel & Distributed DB | 3 zone schemas + DB Links + distributed views + simulated 2-Phase Commit for cross-zone bookings (e.g., train crossing zone boundary) |
| Client-Server Architecture | Strict 3-tier: React client ↔ Express API server ↔ Oracle/Redis/Mongo servers, no direct DB access from client |
| Databases on the Web | REST/JSON API, WebSocket push, XML export endpoint for legacy interop (IRCTC-style) |
| Semi-Structured Data | Oracle `JSON` columns for fare rules/amenities; XMLTYPE column for a legacy "train manifest" export |
| NoSQL Integration | Redis (cache + distributed lock), MongoDB (audit trail, notification templates, analytics events) |
| Extra: Query Optimization | Explain-plan-driven indexing, materialized views for admin dashboards |
| Extra: Concurrency Control | Pessimistic locking for seat allocation + optimistic locking (version column) for profile updates |
| Extra: Data Warehousing (bonus) | Nightly ETL job aggregating bookings into a star-schema reporting mart |

### 1.5 Phase Sequence

```
Phase 0  → Repo, Docker, Architecture Scaffolding
Phase 1  → EER Diagram + Oracle DDL (Specialization/Generalization)
Phase 2  → Distributed Zone Databases (Parallel/Distributed DB)
Phase 3  → PL/SQL Core Business Logic (Procedures, Triggers, Locking)
Phase 4  → Object-Oriented DB Layer (Types, Nested Tables, Methods)
Phase 5  → NoSQL + Semi-Structured Integration (Redis, MongoDB, JSON/XML)
Phase 6  → Backend API Layer (Client-Server, REST/WS)
Phase 7  → Authentication, Authorization & Security Hardening
Phase 8  → Real-Time Seat Map & Notification Pipeline
Phase 9  → Frontend UI (Passenger, TTE, Station Master, Admin views)
Phase 10 → Admin Analytics, Materialized Views & Reporting Mart
Phase 11 → Testing, Documentation & Deployment
```

Each phase below has a ready-to-paste **Opencode Prompt**. Paste them **in order**, one at a time, letting the agent finish and verify before moving to the next. Each prompt is self-contained (re-states stack + prior context) so the agent doesn't need full conversation history.

---

## PART 2 — OPENCODE-READY PROMPTS

---

### PHASE 0 — Repository & Architecture Scaffolding

```
ROLE: You are a senior full-stack architect setting up the foundation for an enterprise
railway reservation system called "RailNexus".

PROJECT CONTEXT:
- This is an academic + portfolio project that must demonstrate advanced DBMS concepts:
  EER modeling, PL/SQL, Object-Oriented DB, Distributed/Parallel DB (simulated railway
  zones), Client-Server architecture, semi-structured data (JSON/XML), and NoSQL
  integration (Redis + MongoDB).
- Final stack: React+TS frontend, Node.js+Express+TS backend, Oracle Database (primary,
  3 schemas simulating 3 railway zones), Redis (cache), MongoDB (semi-structured/audit),
  Docker Compose orchestration.

TASK FOR THIS PHASE:
1. Create a monorepo with this exact structure:
   /railnexus
     /frontend          (React+TS app, empty scaffold for now)
     /backend
       /src
         /api
         /services
         /plsql          (all .sql files: DDL, packages, triggers, types)
         /distributed     (DB link scripts, zone config)
         /nosql           (redis + mongo client wrappers)
         /config
         /middleware
       /tests
     /db
       /zone_north
       /zone_west
       /zone_south
       /shared_ddl
     /docs
       architecture.md
       eer-diagram.md
     docker-compose.yml
     README.md
2. Write `docker-compose.yml` provisioning: Oracle Database Free (23ai) container,
   Redis container, MongoDB container, backend Node service, frontend service.
   Use named volumes for Oracle data persistence. Expose Oracle on 1521, Redis 6379,
   Mongo 27017, backend 4000, frontend 5173.
3. Initialize backend as a TypeScript Node project (Express, node-oracledb, ioredis,
   mongodb driver, dotenv, zod for validation) with strict tsconfig.
4. Initialize frontend as a Vite React-TS app with TailwindCSS configured.
5. Write `/docs/architecture.md` documenting the 3-tier client-server architecture,
   the zone-distribution strategy, and how each ADBMS concept maps to a folder/module
   (use the mapping table I will describe: EER→/db DDL, PL/SQL→/backend/src/plsql,
   Distributed→/backend/src/distributed, OOP-DB→ same plsql folder using Oracle TYPEs,
   NoSQL→/backend/src/nosql).
6. Do NOT write business logic yet. This phase is scaffolding only.
7. Output a summary of every file created and how to run `docker-compose up`.

CONSTRAINTS:
- Use TypeScript everywhere on the backend, strict mode on.
- Do not use ORMs that hide raw SQL (no Prisma/TypeORM) — we need raw PL/SQL calls
  via node-oracledb since demonstrating PL/SQL mastery is a core project requirement.
- Keep secrets in .env.example, never hardcode credentials.
```

---

### PHASE 1 — EER Diagram + Oracle Schema (Specialization/Generalization)

```
ROLE: Senior Oracle Database Architect continuing work on the "RailNexus" railway
booking system (see /docs/architecture.md for full context; Node+Express+TS backend,
React frontend, Oracle primary DB, Redis+Mongo secondary stores).

TASK FOR THIS PHASE — EER MODEL & CORE DDL:
1. First, write `/docs/eer-diagram.md` in Mermaid ER syntax describing the FULL EER
   model with these specialization/generalization hierarchies (document cardinality,
   disjoint/overlapping, and participation constraints for each):
   - PERSON (generalization) → PASSENGER, EMPLOYEE
     - EMPLOYEE (generalization) → TTE_CONDUCTOR, STATION_MASTER, ZONE_ADMIN
   - TRAIN (disjoint generalization) → EXPRESS_TRAIN, SUPERFAST_TRAIN,
     PASSENGER_TRAIN, FREIGHT_TRAIN
   - TICKET (generalization) → CONFIRMED_TICKET, RAC_TICKET, WAITLISTED_TICKET
   - PAYMENT (generalization) → UPI_PAYMENT, CARD_PAYMENT, WALLET_PAYMENT
   - Aggregation entities: ZONE, STATION, ROUTE, TRAIN_SCHEDULE, COACH, SEAT,
     BOOKING, WAITLIST_QUEUE, FARE_RULE, REFUND
2. Implement this EER model as Oracle DDL in `/db/shared_ddl/01_core_schema.sql`
   using the "table-per-subtype with shared primary key" specialization pattern:
   - Supertype table (e.g. PERSON) holds common attributes + a DISCRIMINATOR column
     (person_type CHAR(1) CHECK IN ('P','E')).
   - Subtype tables (PASSENGER, EMPLOYEE) have their PK as FK referencing the
     supertype PK (PERSON_ID), enforcing 1:1 total participation via a trigger or
     FK + NOT NULL.
   - Apply the same pattern to TRAIN and TICKET hierarchies.
3. Add proper constraints: PKs, FKs, CHECK constraints, UNIQUE constraints (e.g. PNR
   format), NOT NULL where mandatory.
4. Add indexing strategy comments (B-tree on FKs, composite index on
   (train_id, journey_date) for the SCHEDULE table since it's the hottest query path).
5. Write a `/db/shared_ddl/02_seed_data.sql` with realistic seed data: 3 zones,
   10 stations, 5 trains (one of each subtype), coaches, seats, 5 sample employees,
   10 sample passengers.
6. Write a short `/docs/schema-notes.md` explaining WHY the table-per-subtype pattern
   was chosen over single-table inheritance (performance, normalization, NULL
   avoidance) — this will be used to justify the design in the academic report.

CONSTRAINTS:
- Target Oracle Database 23ai/21c SQL dialect specifically (not generic ANSI SQL).
- Every subtype table must enforce that a PERSON_ID cannot exist in both PASSENGER
  and EMPLOYEE tables (disjoint constraint) — implement via a trigger that checks
  the discriminator, and explain the alternative (a check constraint on a denormalized
  column) in a code comment.
- Do not write PL/SQL business logic procedures yet — that's Phase 3. This phase is
  schema (DDL) only, plus the disjointness-enforcing triggers described above.
```

---

### PHASE 2 — Distributed Zone Databases (Parallel & Distributed DB Simulation)

```
ROLE: Distributed Database Architect continuing the "RailNexus" project. The core
schema from Phase 1 exists in /db/shared_ddl. Oracle Database is the primary RDBMS.

TASK FOR THIS PHASE — SIMULATE A DISTRIBUTED RAILWAY NETWORK:
1. Design 3 separate Oracle schemas (acting as logically separate "zone databases"):
   ZONE_NORTH, ZONE_WEST, ZONE_SOUTH — write their creation scripts in
   `/db/zone_north/schema.sql`, `/db/zone_west/schema.sql`, `/db/zone_south/schema.sql`.
2. Horizontally partition the TRAIN_SCHEDULE, BOOKING, and SEAT tables by
   ZONE_ID: each zone schema owns only the rows for trains that ORIGINATE in that
   zone. Use Oracle's native `PARTITION BY LIST (zone_id)` on the master tables in the
   shared schema AND, separately, replicate the pattern as truly separate schemas to
   demonstrate both "distributed via partitioning" and "distributed via
   federation" — document the difference in `/docs/distribution-strategy.md`.
3. Create Oracle DATABASE LINKs (`CREATE DATABASE LINK`) so ZONE_NORTH can query
   ZONE_WEST and ZONE_SOUTH data for cross-zone journeys (a train originating in
   North terminating in South must be bookable from any zone).
4. Write a distributed view `ALL_ZONES_TRAIN_SCHEDULE` using `UNION ALL` across the
   DB links so the booking engine can query availability system-wide with one call.
5. Implement a PL/SQL procedure `book_cross_zone_ticket` in
   `/backend/src/plsql/distributed_booking.sql` that simulates a TWO-PHASE COMMIT:
   - Phase 1 (prepare): lock the seat row in the origin zone AND verify capacity in
     the destination zone via the DB link, without committing.
   - Phase 2 (commit/rollback): if both checks pass, commit both; if either fails,
     roll back both and raise a custom exception `ex_cross_zone_booking_failed`.
   - Use `DBMS_TRANSACTION` or an explicit compensating-transaction pattern since
     Oracle XA/2PC across DB links has practical limits — document the real-world
     caveat in a comment and explain the simulated approach clearly.
6. Add a `/backend/src/distributed/zoneRouter.ts` module in the Node backend that
   decides, per API request, which zone schema's connection pool to use based on the
   train's zone_id, demonstrating application-level distributed query routing.

CONSTRAINTS:
- Explicitly comment in the SQL where this models "parallel database" behavior
  (independent zone DBs process bookings concurrently with no shared lock contention
  across zones) vs. "distributed database" behavior (DB links, distributed views,
  cross-node transactions).
- Keep all zone schemas structurally identical (same DDL) — only data differs by
  zone, which is the correct pattern for horizontal fragmentation.
```

---

### PHASE 3 — PL/SQL Core Business Logic (Procedures, Triggers, Row-Level Locking)

```
ROLE: PL/SQL Lead Developer on "RailNexus". Schema (Phase 1) and zone distribution
(Phase 2) already exist.

TASK FOR THIS PHASE — TRANSACTIONAL BOOKING ENGINE IN PL/SQL:
1. Create package `booking_pkg` in `/backend/src/plsql/booking_pkg.sql` with:
   - `PROCEDURE book_seat(p_passenger_id, p_schedule_id, p_coach_id, p_seat_pref,
     OUT p_pnr, OUT p_status)`:
     * Uses `SELECT seat_id FROM SEAT WHERE ... FOR UPDATE SKIP LOCKED` to safely
       claim a seat under concurrent load (row-level locking — critical, must be
       explained in comments as the core concurrency-control mechanism).
     * If no seat available, automatically enrolls the passenger in RAC if RAC
       quota remains, else WAITLIST_QUEUE with the next sequence number.
     * Generates a unique PNR via a sequence + custom format function.
     * Inserts into the correct TICKET subtype table (CONFIRMED/RAC/WAITLISTED).
   - `PROCEDURE cancel_ticket(p_pnr)`:
     * Frees the seat, then calls `promote_waitlist` (below) in the same
       transaction.
   - `PROCEDURE promote_waitlist(p_schedule_id)`:
     * Locks the next WAITLIST_QUEUE row `FOR UPDATE`, promotes to RAC or CONFIRMED
       depending on freed inventory, updates queue positions for everyone behind them.
2. Create triggers in `/backend/src/plsql/triggers.sql`:
   - `AFTER INSERT ON BOOKING` → writes an audit row to `BOOKING_AUDIT` (this table
     will later mirror to MongoDB in Phase 5).
   - `BEFORE UPDATE ON SEAT` → prevents overselling by re-validating capacity even
     under trigger-level mutation (defense in depth beyond the FOR UPDATE lock).
   - `INSTEAD OF UPDATE` trigger on a `TICKET_STATUS_VIEW` view that abstracts the
     three TICKET subtype tables into one updatable view (demonstrates handling
     generalization hierarchies via updatable views).
3. Create `fare_pkg` with `FUNCTION calculate_fare(...)` implementing dynamic pricing
   (distance-based base fare + demand multiplier read from the FARE_RULE JSON column
   — stub the JSON read for now, full implementation happens in Phase 5).
4. Write `/backend/src/plsql/exceptions.sql` defining custom exceptions
   (`ex_seat_unavailable`, `ex_invalid_pnr`, `ex_duplicate_booking`) with
   `PRAGMA EXCEPTION_INIT` and consistent error codes in the -20000 range.
5. Write PL/SQL unit tests using utPLSQL (or plain anonymous-block test scripts if
   utPLSQL isn't available) in `/backend/tests/plsql/booking_pkg.test.sql` covering:
   concurrent booking of the last seat (two sessions racing), successful waitlist
   promotion after cancellation, and PNR uniqueness.

CONSTRAINTS:
- Every procedure must be idempotent-safe against re-execution failures (use
  autonomous transactions ONLY for the audit trigger, never for the core booking
  logic, and explain why in a comment).
- Comment extensively — this PL/SQL will be directly graded/reviewed for the
  DBMS course. Explain WHY `FOR UPDATE SKIP LOCKED` was chosen over a plain
  `FOR UPDATE` (throughput under contention) in a top-of-file comment block.
```

---

### PHASE 4 — Object-Oriented Database Layer

```
ROLE: Oracle Object-Relational specialist on "RailNexus". Core schema and PL/SQL
booking engine already exist (Phases 1–3).

TASK FOR THIS PHASE — OBJECT-ORIENTED DATABASE CONCEPTS:
1. In `/backend/src/plsql/object_types.sql`, define Oracle object types:
   - `CREATE TYPE address_t AS OBJECT (line1 VARCHAR2(100), city VARCHAR2(50),
     state VARCHAR2(50), pincode VARCHAR2(10), MEMBER FUNCTION full_address
     RETURN VARCHAR2);` with the body implementing string concatenation.
   - `CREATE TYPE journey_segment_t AS OBJECT (from_station VARCHAR2(10),
     to_station VARCHAR2(10), distance_km NUMBER, MEMBER FUNCTION segment_fare
     RETURN NUMBER);` — a reusable object representing one leg of a multi-leg
     journey.
   - `CREATE TYPE journey_segment_list_t AS TABLE OF journey_segment_t;` (a nested
     table collection type) to represent a train's full multi-station route as a
     single column value.
   - `CREATE TYPE emergency_contact_varray_t AS VARRAY(3) OF VARCHAR2(15);` for a
     passenger's up-to-3 emergency contact numbers.
2. Alter the PERSON table to add an `address address_t` column (embedded object)
   and the PASSENGER table to add `emergency_contacts emergency_contact_varray_t`.
3. Alter ROUTE table to add `segments journey_segment_list_t` (nested table column)
   with a proper `NESTED TABLE segments STORE AS route_segments_tab` clause,
   demonstrating Oracle's object-relational nested-table storage.
4. Write example DML showing how to INSERT/SELECT/query these object-typed columns,
   including a `SELECT ... FROM TABLE(r.segments)` unnesting query to list all legs
   of a route, saved in `/backend/src/plsql/oop_examples.sql`.
5. Write a PL/SQL function `total_route_distance(p_route_id) RETURN NUMBER` that
   iterates the nested table collection with a `FOR i IN 1..segments.COUNT` loop and
   sums distances, demonstrating object collection iteration.
6. Document in `/docs/oop-db-notes.md` how this differs from the pure relational
   approach used elsewhere in the schema, and why object types were used ONLY for
   ROUTE segments and ADDRESS/CONTACTS (naturally hierarchical/repeating data) rather
   than throughout — justify the selective use as good design, not overuse.

CONSTRAINTS:
- Do not replace the core relational tables from Phase 1 — this phase augments them
  with object-relational features, it does not redesign the whole schema as pure OOP.
- Every object type needs at least one MEMBER FUNCTION/PROCEDURE to genuinely
  demonstrate OOP behavior (methods bound to data), not just structs.
```

---

### PHASE 5 — NoSQL & Semi-Structured Data Integration

```
ROLE: Polyglot-persistence engineer on "RailNexus". Oracle schema, PL/SQL engine,
and object types already exist (Phases 1–4). Redis and MongoDB containers are already
running via docker-compose (Phase 0).

TASK FOR THIS PHASE:
1. SEMI-STRUCTURED DATA IN ORACLE:
   - Add a `fare_rules JSON` column to FARE_RULE table (native Oracle 21c+ JSON
     type) storing dynamic pricing rules like
     `{"base_multiplier":1.0,"surge":{"weekend":1.2,"festival":1.5},
     "class_multipliers":{"1A":3.0,"2A":2.0,"SL":1.0}}`.
   - Update `fare_pkg.calculate_fare` (from Phase 3) to actually parse this JSON
     using `JSON_VALUE`/`JSON_TABLE` to compute the final fare.
   - Add an `XMLTYPE` column `manifest_xml` on TRAIN_SCHEDULE representing a legacy
     "train manifest" export format (passenger list, coach layout) and write a
     PL/SQL function `generate_manifest_xml(p_schedule_id) RETURN XMLTYPE` using
     `DBMS_XMLGEN` or `XMLELEMENT`/`XMLAGG` to build it from relational data.
2. REDIS INTEGRATION (`/backend/src/nosql/redisClient.ts`):
   - Cache-aside pattern for seat availability: `GET zone:{zoneId}:schedule:
     {scheduleId}:availability`, populated from Oracle on cache miss, TTL 30s.
   - Implement a Redis-based DISTRIBUTED LOCK (`SET key value NX PX 5000`) used as
     a first-line guard before hitting the Oracle `FOR UPDATE` row lock, to reduce
     DB contention under high concurrent demand (explain in comments: Redis lock =
     fast-fail at app tier, Oracle lock = source of truth).
   - Session store for JWT refresh tokens with TTL matching token expiry.
3. MONGODB INTEGRATION (`/backend/src/nosql/mongoClient.ts`):
   - `booking_audit_log` collection: every booking/cancel/promotion event from the
     Oracle `BOOKING_AUDIT` table (Phase 3 trigger) is also written here as a rich,
     schema-flexible JSON document (include full snapshot of ticket + passenger +
     fare breakdown at time of event) — implement the dual-write from the Node
     service layer, and explain in comments the eventual-consistency trade-off vs.
     writing only to Oracle.
   - `notification_templates` collection storing semi-structured, varying-shape
     templates per notification type (SMS/email/push) — since template structure
     varies by channel, this is a genuine NoSQL fit (document why relationally this
     would need many nullable columns or EAV, and why Mongo is cleaner here).
   - `analytics_events` collection for clickstream/search events (high write volume,
     flexible schema) — explicitly contrasted against Oracle OLTP tables in
     `/docs/nosql-justification.md`.
4. Write `/docs/nosql-justification.md` explaining, per NoSQL use case above, WHY it
   was chosen over Oracle (write volume, schema flexibility, TTL/cache semantics) —
   this is the academic justification for polyglot persistence.

CONSTRAINTS:
- Redis and Mongo must never be the source of truth for financial/seat-state data —
  Oracle remains authoritative; NoSQL stores are caches, logs, or genuinely
  schema-flexible auxiliary data only. State this explicitly in code comments.
```

---

### PHASE 6 — Backend API Layer (Client-Server Architecture)

```
ROLE: Backend Lead Engineer on "RailNexus". All database layers (Oracle PL/SQL,
distributed zones, object types, Redis, MongoDB) exist from Phases 1–5. Now expose
them via a proper API.

TASK FOR THIS PHASE:
1. Build a clean layered Express+TS API in `/backend/src/api`:
   - `routes/` (thin HTTP routing only)
   - `controllers/` (request validation via zod, response shaping)
   - `services/` (business orchestration — calls PL/SQL procedures via
     node-oracledb, checks Redis cache, writes Mongo audit)
   - `repositories/` (raw node-oracledb calls to PL/SQL packages — the ONLY layer
     allowed to touch the DB driver directly)
2. Implement these REST endpoints (document them in an OpenAPI spec at
   `/backend/openapi.yaml`):
   - `GET /api/trains/search?from&to&date` → queries `ALL_ZONES_TRAIN_SCHEDULE`
     (Phase 2 distributed view), checks Redis cache first.
   - `POST /api/bookings` → calls `booking_pkg.book_seat`, returns PNR + status.
   - `DELETE /api/bookings/:pnr` → calls `booking_pkg.cancel_ticket`.
   - `GET /api/bookings/:pnr` → status lookup, unions CONFIRMED/RAC/WAITLISTED via
     the `TICKET_STATUS_VIEW` from Phase 3.
   - `GET /api/schedules/:id/manifest.xml` → returns the XMLTYPE manifest
     (Phase 5) with `Content-Type: application/xml` — this is the "Databases on
     the Web + semi-structured export" endpoint.
   - `GET /api/schedules/:id/fare-rules` → returns the raw JSON fare rule document.
3. Enforce the client-server boundary: the React frontend (Phase 9) will NEVER
   connect to Oracle/Redis/Mongo directly — only through this API. Add a middleware
   that rejects any request without a valid API key/JWT (full auth is Phase 7, for
   now stub a simple API-key header check).
4. Implement centralized error handling that translates Oracle custom exceptions
   (ex_seat_unavailable etc. from Phase 3) into proper HTTP status codes (409, 404,
   422) with a consistent JSON error shape `{code, message, details}`.
5. Add connection pooling config for node-oracledb (min/max pool size, per-zone
   pools from the Phase 2 zoneRouter) and graceful shutdown handling.
6. Write integration tests with Jest+Supertest for the booking and cancellation
   flow, including a concurrency test that fires 20 simultaneous booking requests
   for a schedule with 5 seats left and asserts exactly 5 succeed and 15 are
   waitlisted/RAC (validates the Phase 3 row-locking end-to-end through the API).

CONSTRAINTS:
- No business logic in controllers — controllers only validate input and call
  services. All transactional logic stays in PL/SQL (called from repositories).
- Response times for the search endpoint must use the Redis cache-aside pattern
  from Phase 5 — do not query Oracle on every search request.
```

---

### PHASE 7 — Authentication, Authorization & Security Hardening

```
ROLE: Application Security Engineer on "RailNexus". API layer exists (Phase 6).

TASK FOR THIS PHASE:
1. Implement JWT auth in `/backend/src/api/auth`:
   - `POST /api/auth/register` (passenger self-signup), `POST /api/auth/login`
     (passengers + employees, role resolved from the PERSON/EMPLOYEE
     specialization tables from Phase 1).
   - Access token (15 min) + refresh token (7 days, stored in Redis per Phase 5,
     rotated on use).
   - Passwords hashed with bcrypt (cost factor 12), never logged.
2. Implement RBAC middleware mapping roles to endpoint permissions:
   - PASSENGER: search, book, cancel, view own bookings.
   - TTE_CONDUCTOR: view passenger manifest for their assigned train
     (calls the Phase 5 manifest XML endpoint), mark seat no-show.
   - STATION_MASTER: view all bookings for their station, trigger platform
     announcements.
   - ZONE_ADMIN: full CRUD on trains/schedules within their zone only (enforce
     zone-scoping using the Phase 2 zone router).
3. Add security middleware: helmet, rate limiting (per-IP and per-user, stricter on
   /auth endpoints), input sanitization against SQL injection (parameterized
   node-oracledb binds ONLY — audit every repository call from Phase 6 to confirm
   no string-concatenated SQL exists anywhere), CORS locked to the frontend origin.
4. Add row-level authorization checks in the service layer so a passenger can never
   fetch another passenger's PNR details even with a guessed ID (IDOR protection).
5. Write an audit log (to the Mongo `booking_audit_log`/new `security_audit_log`
   collection from Phase 5) for every failed login, permission-denied event, and
   admin action.
6. Write security-focused tests: expired token rejection, role-escalation attempt
   rejection, SQL-injection-payload rejection, rate-limit trigger.

CONSTRAINTS:
- Never store plaintext passwords or full card numbers anywhere (payments,
  Phase 3's PAYMENT subtype tables, should store only masked/tokenized references).
- Every new endpoint from this phase forward must declare its required role(s)
  explicitly in the route definition — no endpoint may be left unauthenticated
  except /auth/login and /auth/register and the public train-search endpoint.
```

---

### PHASE 8 — Real-Time Seat Map & Notification Pipeline

```
ROLE: Real-time systems engineer on "RailNexus". Secure API layer exists (Phase 7).

TASK FOR THIS PHASE:
1. Add a Socket.IO gateway in `/backend/src/api/realtime`:
   - Clients subscribe to `schedule:{id}:seatmap` for a given train/date.
   - On every successful `booking_pkg.book_seat` or `cancel_ticket` call (Phase 3),
     the service layer emits a `seat-updated` event with the changed seat's new
     status, sourced from the same Redis cache invalidation used in Phase 5 (cache
     invalidate-then-refresh-then-broadcast pattern).
2. Add a notification pipeline:
   - On booking confirmation, RAC/waitlist promotion, or cancellation, enqueue a
     notification job (use a simple in-memory queue or BullMQ backed by the
     existing Redis instance) that renders the appropriate
     `notification_templates` document (Phase 5 Mongo collection) and "sends" it
     (stub the actual SMS/email provider with a logged mock adapter, but build the
     full pipeline so a real provider can be swapped in).
3. Add a `GET /api/schedules/:id/seatmap` REST endpoint for initial page load
   (Socket.IO only pushes deltas after that).
4. Write a load-test script (k6 or simple Node script) simulating 100 concurrent
   users watching one seat map while 10 users book seats, verifying all clients
   receive consistent, ordered updates.

CONSTRAINTS:
- The Socket.IO layer must only ever broadcast state that originated from a
  successful, committed Oracle transaction — never broadcast optimistic/unconfirmed
  state.
```

---

### PHASE 9 — Frontend UI

```
ROLE: Senior Frontend Engineer on "RailNexus". Full backend (API, auth, realtime)
exists from Phases 1–8. OpenAPI spec is at /backend/openapi.yaml.

TASK FOR THIS PHASE:
1. Scaffold the React+TS frontend (Vite, TailwindCSS, Redux Toolkit + RTK Query
   generated from the OpenAPI spec, React Router) with these route groups:
   - Public: Landing page, train search results, login/register.
   - Passenger (protected): booking flow (search → select → passenger details →
     payment mock → confirmation with live PNR status), My Bookings, cancellation.
   - TTE (protected, role-gated): assigned-train manifest view (renders data from
     the XML manifest endpoint, Phase 6), mark-boarded action.
   - Station Master (protected): station bookings dashboard, announcement trigger.
   - Zone Admin (protected): train/schedule CRUD scoped to their zone.
2. Build a live seat-map component that connects to the Socket.IO gateway (Phase 8)
   and visually updates seat colors (available/RAC/confirmed/locked) in real time.
3. Build a booking-status tracker component showing waitlist position moving in
   real time as others cancel.
4. Implement proper loading/error/empty states for every data-fetching view, and a
   global toast system for booking success/failure/notification events.
5. Add accessible, responsive design (mobile-first for passengers, dense
   data-table-first for admin/TTE roles).

CONSTRAINTS:
- The frontend must call ONLY the backend REST/WS API — never any database driver
  directly (enforces the client-server boundary from Phase 6).
- Use RTK Query for all server-state; do not hand-roll fetch+useState data fetching.
```

---

### PHASE 10 — Admin Analytics, Materialized Views & Reporting Mart

```
ROLE: Data/Analytics engineer on "RailNexus". OLTP system (Phases 1–9) is fully
functional.

TASK FOR THIS PHASE:
1. In Oracle, create materialized views for the admin dashboard:
   - `MV_DAILY_OCCUPANCY` (per train/date: seats sold, RAC count, waitlist count,
     revenue) with `REFRESH FAST ON COMMIT` using materialized view logs on
     BOOKING — demonstrating incremental refresh, not full rebuild.
   - `MV_ZONE_REVENUE_SUMMARY` aggregating across the 3 zone schemas via the
     Phase 2 DB links, `REFRESH COMPLETE ON DEMAND` nightly.
2. Design a lightweight star schema in a new `REPORTING` schema:
   - Fact table `FCT_BOOKING` (grain: one row per ticket) with FKs to
     `DIM_DATE`, `DIM_TRAIN`, `DIM_STATION`, `DIM_ZONE`, `DIM_PASSENGER_SEGMENT`.
   - Write a PL/SQL ETL procedure `etl_load_reporting_mart` that incrementally
     loads new bookings since the last run (watermark-based) from OLTP tables into
     the fact/dim tables — schedule it conceptually via `DBMS_SCHEDULER` (write the
     job creation script even if it won't run automatically in dev).
3. Build backend endpoints + a frontend admin analytics dashboard (charts: revenue
   trend, occupancy heatmap by route, top waitlisted routes) reading from the
   materialized views / reporting mart — never from live OLTP tables, to avoid
   analytical queries degrading booking performance.
4. Document in `/docs/reporting-architecture.md` the OLTP-vs-OLAP separation
   rationale (this is a strong academic talking point: workload isolation).

CONSTRAINTS:
- Analytics queries must never run against the live BOOKING/SEAT tables directly —
  always against materialized views or the reporting mart.
```

---

### PHASE 11 — Testing, Documentation & Deployment

```
ROLE: DevOps/QA Lead closing out the "RailNexus" project. All features from
Phases 1–10 are implemented.

TASK FOR THIS PHASE:
1. Achieve test coverage across layers:
   - PL/SQL: utPLSQL suite covering every procedure in booking_pkg, fare_pkg,
     waitlist promotion, and the distributed 2PC simulation from Phase 2.
   - Backend: Jest unit tests for services, Supertest integration tests for every
     endpoint, plus the concurrency test from Phase 6.
   - Frontend: Playwright E2E covering the full passenger booking journey, TTE
     manifest view, and admin CRUD flow.
2. Write final documentation:
   - `/docs/architecture.md` — update with final diagrams (system, deployment,
     sequence diagram for the booking transaction showing Redis lock → Oracle
     FOR UPDATE → trigger → Mongo audit write → Socket.IO broadcast).
   - `/docs/adbms-concepts-mapping.md` — a single reference table mapping every
     ADBMS syllabus topic to the exact file/procedure that demonstrates it, with
     line-number references, for direct use in the academic report/viva.
   - `/README.md` — full setup instructions (`docker-compose up`, seed data load,
     default logins per role).
3. Finalize `docker-compose.prod.yml` with health checks, restart policies, and
   environment-based config separation from dev.
4. Add a GitHub Actions CI workflow running lint, backend tests, and a
   docker-compose smoke test on every push.

CONSTRAINTS:
- Do not mark this phase complete until `/docs/adbms-concepts-mapping.md` cleanly
  lists all 8+ ADBMS concepts with concrete file references — this document is the
  primary academic deliverable artifact.
```

---
