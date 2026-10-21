# Architecture — Budget 2.0

| Field | Value |
|-------|-------|
| **Status** | **PROPOSED** — awaiting owner acceptance of this file as the architecture baseline |
| **Source issue** | [#2 — Decide on what architecture to use for this project](https://github.com/zachkingcade/ZK-Dynamic-Flow-Budgeting-Application/issues/2) |
| **Last updated** | 2026-10-16 |
| **Scope** | Runtime topology, Clean Architecture layering, module map, stack, Docker Compose, event-driven model, multi-user / multi-budget data shape, Angular UI placement |
| **Companion** | Domain / financial rules live in [`docs/BUSINESS_RULES.md`](./BUSINESS_RULES.md) (**Accepted**). This document does **not** restate those rules; it defines how the system is structured to uphold them. |

### Status legend

| Label | Meaning |
|-------|---------|
| **ACCEPTED** | Owner-decided; implementations must obey once this file is human-accepted |
| **PROPOSED** | Team recommendation locked from Issue #2 consultation; not binding until the owner promotes this file |
| **OPEN** | Still needs an explicit human decision |
| **DEFERRED** | Intentionally out of scope for this document / later issue |

---

## 1. Purpose and relationship to business rules

### Purpose

Define the **software architecture** for Budget 2.0 so a small team can implement a hosted, Docker Compose–deployed application that:

- Preserves financial integrity defined in `BUSINESS_RULES.md` (conservation, atomic reject, funding DAG, double-entry).
- Stays operable as a **modular monolith** with clear internal boundaries.
- Supports **multiple users** (accounts) and **multiple budgets per user**, with **at most one user per account**.
- Uses an **event-driven** application model (no wall-clock / cron-driven automatic funding as the primary design).

### Non-goals of this document

- Rewriting or amending business rules (→ `BUSINESS_RULES.md`).
- Full IAM / authentication / authorization design (**DEFERRED** — later issue; Security owns).
- UI visual design, branding, or Angular component library choices.
- Bank sync, multi-currency, multi-user collaboration on one budget, tax/legal advice (product non-goals per business rules).
- Microservices, message brokers, or multi-database topologies for v1.

---

## 2. Decision summary (ADR-style) — Issue #2

Owner-locked decisions for Issue #2:

| ID | Decision | Choice | Status |
|----|----------|--------|--------|
| D1 | Runtime topology | **Modular monolith** — one Spring Boot deployable for the money path; clear internal modules | PROPOSED → accept with this file |
| D2 | Application style | **Clean Architecture** (domain center; application use-cases; inbound/outbound adapters) | PROPOSED → accept with this file |
| D3 | Backend stack | **Java / Spring Boot** | PROPOSED → accept with this file |
| D4 | Frontend | **Angular** web app — part of the product monolith conceptually; **own section** in this doc; separate build artifact | PROPOSED → accept with this file |
| D5 | Persistence | **PostgreSQL** + schema versioning with **Flyway** | PROPOSED → accept with this file |
| D6 | Driving model | **Event-driven** — no real-time wall-clock scheduler / cron as the primary automatic-funding trigger | PROPOSED → accept with this file |
| D7 | Tenancy | Software + DB support **many accounts/users**; **≤1 user per account**; each user may have **many budgets** | PROPOSED → accept with this file |
| D8 | Hosting | **Public host from the start** via Docker Compose (+ reverse proxy) | PROPOSED → accept with this file |
| D9 | Auth | **Designed later** — placeholder only in v1 architecture | DEFERRED |
| D10 | Module enforcement | **Convention-only** for now (no import-lint / ArchUnit required yet) | PROPOSED → accept with this file |

### Axes (do not conflate)

| Axis | What we decided |
|------|-----------------|
| **Deployment / runtime** | Modular monolith in Compose (`api`, `web`, `db`) — not microservices |
| **Code organization** | Clean Architecture layers inside modules |
| **Stack** | Spring Boot + Angular + PostgreSQL + Flyway |
| **Reaction model** | Domain / application events and explicit commands — not cron |

---

## 3. Runtime topology

### 3.1 Modular monolith — PROPOSED (pending file acceptance)

**One** Spring Boot application process owns:

- HTTP API
- Domain model and use-cases
- Persistence adapters (PostgreSQL)
- In-process domain / application event publishing and handling

**Rationale (summary):**

- `BUSINESS_RULES.md` requires **atomic funding preview + commit** (multi-parent / recursive pulls with the triggering transaction) and **reject ⇒ zero change**. That maps cleanly to **one database transaction** in one process.
- Single economic pool per budget and conservation checks favor a **single write model**.
- Docker Compose hosting stays simple; evolvability comes from **module boundaries**, not network splits.

**Not in v1:** separate funding-service / ledger-service, Kafka (or equivalent) as a required backbone, or per-module databases.

### 3.2 Logical deployables

| Deployable | Technology | Role |
|------------|------------|------|
| **API** | Spring Boot JAR | Domain, use-cases, REST (and later auth), event handling |
| **Web** | Angular → static assets behind nginx (or Caddy) | Browser UI; calls API; TLS + reverse proxy in v1 |
| **DB** | PostgreSQL | System of record |

Angular is **not** a separate microservice. It is a **separate build and container** that forms one product with the API.

---

## 4. Clean Architecture (within the modular monolith)

### 4.1 Layers

Dependency rule: **inward only**. Domain depends on nothing in the framework. Application depends on domain. Adapters depend on application/domain abstractions (ports).

```text
┌─────────────────────────────────────────────────────────┐
│  Adapters (inbound)                                      │
│  REST controllers, event listeners, (future) auth filter │
└──────────────────────────┬──────────────────────────────┘
                           │ calls
┌──────────────────────────▼──────────────────────────────┐
│  Application (use-cases / ports)                         │
│  Orchestrate preview → validate → commit; publish events │
└──────────────────────────┬──────────────────────────────┘
                           │ uses
┌──────────────────────────▼──────────────────────────────┐
│  Domain                                                  │
│  Entities, value objects, domain services, domain events │
│  Invariants aligned with BUSINESS_RULES.md               │
└──────────────────────────┬──────────────────────────────┘
                           ▲ implemented by
┌──────────────────────────┴──────────────────────────────┐
│  Adapters (outbound)                                     │
│  JPA/JDBC repositories, clock/timezone, Flyway-managed   │
│  schema, (optional) mail/metrics sinks                   │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Spring Boot packaging convention

Base package (illustrative): `com.zk.budget`.

**Prefer package-by-module, then layer** (modular monolith clarity):

```text
com.zk.budget
├── bootstrap/                    # @SpringBootApplication, wiring, security placeholder config
├── shared/                       # cross-cutting primitives (Money/MinorUnits, ids, Result/rejection types)
│   ├── domain/
│   └── application/
├── identity/                     # Account + User placeholders (auth DEFERRED)
│   ├── domain/
│   ├── application/
│   └── adapter/
│       ├── inbound/web/
│       └── outbound/persistence/
├── budgetstructure/              # Budget, nodes, funding edges, request strategies, targets/max config
│   ├── domain/
│   ├── application/
│   └── adapter/...
├── ledger/                       # Journals, Available, BudgetPoolCash, conservation
│   ├── domain/
│   ├── application/
│   └── adapter/...
├── funding/                      # Funding planner (preview), auto-fund policy application, reclaim
│   ├── domain/
│   ├── application/
│   └── adapter/...
├── periods/                      # Period cadence, open/close, timezone interpretation gates
│   ├── domain/
│   ├── application/
│   └── adapter/...
└── reporting/                    # Targets variance, read models (no reject authority)
    ├── domain/                   # keep thin
    ├── application/
    └── adapter/...
```

**Layer contents (convention):**

| Layer | Contains | Must not contain |
|-------|----------|------------------|
| `domain` | Entities, VOs, domain services (e.g. DAG cycle check, split/renormalize math), domain events, invariant guards | Spring Web, JPA annotations *preferred avoided*; no REST DTOs |
| `application` | Use-case services, inbound/outbound **ports** (interfaces), transaction boundary policies, application events | SQL, HTTP types |
| `adapter.inbound.web` | Controllers, request/response DTOs, exception → HTTP mapping | Business rules |
| `adapter.inbound.events` | `@TransactionalEventListener` / handlers that call use-cases | Direct balance mutation bypassing use-cases |
| `adapter.outbound.persistence` | JPA entities/repositories implementing ports, mappers to domain | Funding policy decisions |
| `bootstrap` | Composition root only | Domain logic |

**Convention-only enforcement (D10):** document and code-review these boundaries. ArchUnit / import-lint may be added later; not required for v1. Module conventions are **not** a security control — ownership checks belong in use-cases and persistence.

### 4.3 Money path use-case shape (normative)

For any path that can move money or reject (external expense/income, automatic funding application, manual move, reclaim):

1. **Load** persisted config + balances for the budget (consistent read).
2. **Plan** automatic funding if required (`Funding` domain/application) — atomic preview.
3. If uncoverable and Allow deficit is false → **reject**; commit nothing.
4. If acceptable → **commit** funding legs + triggering journal entries in **one DB transaction**.
5. On success → **publish domain/application events** (audit, projections). Prefer after-commit listeners for non-critical side effects.

**Event-driven does not mean eventually consistent money commits.** Funding legs and the triggering external transaction must not be split across async consumers.

Clients may pre-check for UX; **domain re-evaluates** from persisted state (business rule).

---

## 5. Domain module map (aligned with BUSINESS_RULES.md)

Modules are logical. They share one Postgres database and one Spring process.

| Module | Responsibility | Upholding (pointer) |
|--------|----------------|---------------------|
| **Identity** | Account, User (1 user ↔ 1 account), ownership of budgets | Tenancy D7; auth DEFERRED |
| **Budget structure** | Budgets, nodes (pool vs spending), funding edges (DAG), baselines, hard max, targets, request strategies, Allow deficit flags | Topology, config rejects (cycles, target ≤ max) |
| **Ledger** | Double-entry journals, Available, BudgetPoolCash, `{Node} Expense` accounts, conservation | Financial integrity |
| **Funding** | Preview/plan multi-parent & recursive pulls; apply reclaim; distinguish automatic vs manual moves | Atomic preview; keep-money-high policy in planner |
| **Periods** | Timezone, cadence metadata, open/close commands, transaction-date gate vs open period | Period lifecycle |
| **Reporting** | Target variance / over-under; never alone accepts/rejects posts | Target is variance-only |

**Transfers vs calculated allocations:** Automatic funding **calculates a plan** from graph + rules, then **executes durable zero-sum ledger posts**. Available is not “derived at read time” from rules; the ledger is system of record.

**Cycles:** Config path rejects cycle-creating edges; runtime funding **fail-closed** on cycle detection (domain).

**Multi-budget isolation:** Each budget is its own economic pool (`Σ Available = BudgetPoolCash` per budget). No shared cash, hard-max counters, or period funded-in across budgets.

---

## 6. Event-driven model (no wall-clock scheduler)

### 6.1 Decision

**Owner-locked:** Nothing is scheduled in real time. **Automatic funding and period transitions are not driven by cron or in-process timers** as the primary model.

Triggers are **explicit commands** and **domain/application events**.

### 6.2 Why not async for the commit path

Atomic reject and conservation require that **transaction-time funding** run **inside the same use-case transaction** as the expense/income post — not via an eventually consistent queue consumer.

**Events are the integration style inside the monolith**, not an excuse to split the money commit across consumers.

### 6.3 Recommended mechanism (v1)

| Mechanism | Use |
|-----------|-----|
| **Synchronous use-case orchestration** | All balance-changing operations |
| **Spring domain events** (record types raised by aggregates/domain services) | Express “what happened” in domain language |
| **Spring application events** + `@TransactionalEventListener(phase = AFTER_COMMIT)` | Audit append, read-model updates, non-money side effects |
| **Transactional outbox (optional, same DB)** | Only if a side effect must be reliably processed asynchronously later; **not** required to start |
| **Kafka / Rabbit / separate bus** | **Out of scope for v1** — unjustified for a modular monolith |

Event payloads carry IDs and amounts needed for handlers — not credentials, tokens, or full PII. Treat payloads as sensitive financial data in logs. Money-related handlers must be **idempotent**; events are not a trust boundary and must not bypass ownership checks.

### 6.4 Mapping business triggers → events/commands

| Business trigger (`BUSINESS_RULES`) | Architectural interpretation |
|-------------------------------------|------------------------------|
| **Transaction-time automatic funding** | Part of `PostExternalTransaction` (or equivalent) use-case: preview → commit together. May emit `ExternalTransactionPosted`, `AutomaticFundingApplied`, or `TransactionRejected` after outcome. |
| **Period open** (user closes previous / opens next — user-driven lifecycle) | Explicit command e.g. `OpenPeriod` / `ClosePeriod`. On successful open → domain event `PeriodOpened`. |
| **Scheduled / fixed automatic funding at period open** | Runs **inside `OpenPeriod`’s unit of work** (same DB transaction / orchestrated command): evaluate “on period open” and “every X periods” against period sequence, apply funding plans transactionally with **idempotency** (see §6.5). Failures are visible to the user action. Async outbox for this path is not the v1 default. |
| **Every X periods** | **Not a timer.** Evaluated when a period lifecycle command runs (`OpenPeriod` / `PeriodOpened` path): compare period ordinal/sequence to rule config; if due, apply; record intent for **idempotency**. |
| **Manual fund move / reclaim** | Explicit commands; emit audit events after commit. |
| **Config changes** (max, target, edges) | Commands with validation; emit audited `RuleChanged`-style events; no silent rewrite of history. |

Removing wall-clock schedulers must **not** remove period-open baseline funding; only the trigger source changes (user/domain command instead of cron).

### 6.5 Idempotency

Period-open auto-fund and client retries must be idempotent (business scenario W6). Persist an **intent / application record** (e.g. rule id + period id + kind) in the same DB, unique-constrained, checked inside the funding application transaction.

### 6.6 Concurrency

Concurrent posts that pull from the same funder must not oversubscribe Available or remaining hard max. Enforce with **PostgreSQL transactions** plus appropriate locking (row locks on balance / period-funded-in rows, or equivalent optimistic versioning with retry). Details → later design/ADR; architecture requires a **single-DB locking story**, not distributed locks.

---

## 7. Multi-user and multi-budget data shape

### 7.1 Tenancy model — PROPOSED (pending file acceptance)

```text
Account 1 ──────── 1 User          (never more than one user per account)
Account 1 ──────── * Budget
Budget   1 ──────── * Node, Edge, Period, Journal, …  (budget-scoped)
```

| Concept | Rule |
|---------|------|
| **Account** | Tenancy root for credentials/profile (auth fields DEFERRED) |
| **User** | Exactly one human principal per account in v1 |
| **Budget** | Owned by one account/user; a user may have many budgets |
| **Isolation** | All money graphs and ledgers are **per budget**; no shared pool across budgets |

### 7.2 Schema expectations (high level) — security baseline

Ownership is a **persistence invariant**, not an auth feature. Auth later proves identity; it does not invent tenancy.

- Every budget-scoped table carries `budget_id`.
- Budget (and other tenant roots as needed) carry a stable **owner** reference (`owner_user_id` or equivalent FK to the account/user).
- All reads/writes are scoped by **owner + budget_id**; never trust client-supplied ownership alone (prevents IDOR when auth arrives).
- Flyway migrations own all schema changes; no ad-hoc prod DDL.
- No “global” financial tables without owner/budget foreign keys.

### 7.3 Note on business-rules non-goal wording

`BUSINESS_RULES.md` lists “Multi-user capability / IAM design” as a **product non-goal for v1 rules scope**. This architecture **does** require a **data model for multiple accounts/users and multiple budgets**, while **IAM/authz design remains deferred**. That is intentional: multi-tenant *data shape* now; *security model* later.

---

## 8. Stack

| Layer | Choice | Notes |
|-------|--------|-------|
| API | Java, Spring Boot | Clean Architecture packages as in §4 |
| UI | Angular | SPA; §9 |
| DB | PostgreSQL | System of record |
| Migrations | Flyway | Versioned SQL (or Java) migrations in the API build |
| API style | REST JSON (default assumption) | Exact resource design OPEN |
| Money | Integer **minor units** in domain and DB | Per business rules |

Stack choices beyond this table (exact Spring/Angular major versions, build tool, etc.) may be pinned in implementation setup issues.

---

## 9. UI architecture — Angular

### 9.1 Placement

- Angular is the **browser client** for the modular monolith product.
- It talks to the Spring Boot API over HTTP(S) (JSON).
- It **must not** enforce financial invariants alone; server domain re-validates (business rule).

### 9.2 Responsibilities

| UI may | UI must not |
|--------|-------------|
| Capture user intent (post expense, open/close period, edit graph) | Be the only place that checks hard max / deficit / cycles |
| Optimistic UX / client-side previews labeled as estimates | Commit money locally |
| Present rejects and audit explanations from API | Bypass API with direct DB access |

### 9.3 Codebase layout (convention)

Monorepo (recommended default):

```text
/
├── apps/
│   └── api/          # Spring Boot
│   └── web/          # Angular
├── docs/
└── docker-compose.yml
```

(Exact folder names OPEN; principle = separate app modules, one repo.)

### 9.4 Serving in production (v1)

**Chosen default:** nginx (or Caddy) container serves the Angular **production build** and reverse-proxies API routes. One `web` service combines SPA + TLS + proxy (no separate proxy container required for v1).

```text
Browser
   │
   ▼
┌──────────────────────────────┐
│  web (nginx/Caddy)           │
│  TLS                         │
│  /        → Angular SPA      │
│  /api/*   → api:8080         │
└──────────────┬───────────────┘
               │
               ▼
        ┌─────────────┐     ┌────────────┐
        │ api (Spring)│────▶│ db (Postgres)│
        └─────────────┘     └────────────┘
```

**Not v1 default:** embedding Angular into the Spring JAR; separate public CDN without proxy before auth exists.

SPA builds receive **public** config only (API base URL, future public client id). No private keys or DB URLs in Angular artifacts.

---

## 10. Docker Compose deployment (public host)

### 10.1 Services (v1)

| Service | Image / build | Purpose |
|---------|---------------|---------|
| `db` | Official PostgreSQL | Data at rest (internal network only) |
| `api` | Build from Spring Boot | Application + Flyway migrate on startup (or init job — OPEN) |
| `web` | Build Angular → nginx/Caddy image | SPA + reverse proxy + TLS |

### 10.2 Public exposure — mandatory baseline

Public hosting is an **infrastructure** choice; authentication is a **product gate**.

- Only `web` publishes host ports 80/443.
- `api` and `db` stay on the internal Compose network.
- **Postgres must not** be published to `0.0.0.0` / the public internet.
- Secrets (DB password, future JWT/OIDC keys) via env or Compose secrets — **not** baked into images or committed to git.
- CORS / origin policy locked to the SPA origin (detail may wait; principle cannot).
- TLS termination on the public path before any credentials exist.

**.env / secret files** are gitignored; provide `.env.example` with dummy values only.

### 10.3 Auth placeholder (public host risk)

Until auth is designed:

- The deployment **may** be reachable on the public internet.
- Architecture assumes **all mutating and financial-read API calls will eventually be authenticated** to an Account/User and authorized to a Budget.
- **Do not** ship an open financial API as a permanent state. “Auth deferred” does **not** mean “open API on the internet.” Until the Auth issue lands, early deploys must use an **approved temporary control** (e.g. VPN, IP allowlist, private bind, or delayed public DNS) — **Security** defines the real control in a later issue.
- Clean Architecture: when auth arrives, it sits at the API/application edge; domain stays free of framework auth types, but **use cases enforce ownership**.

This document does **not** specify OAuth, sessions, JWT, password hashing, or roles.

Placeholders for future externalized config (names only): e.g. `JWT_ISSUER`, `JWT_AUDIENCE`, `OIDC_*` / signing-key refs.

---

## 11. Persistence and consistency

| Topic | Architecture stance |
|-------|---------------------|
| System of record | PostgreSQL via outbound persistence adapters |
| Migrations | Flyway, applied by API (or documented init) |
| Transaction boundary | One TX per successful money use-case (funding legs + external post) |
| Reject | No writes to balances / period funded-in / journals for that intent |
| Audit | Append-only store preferred; can start as tables written in-adapter after domain decision |
| Read models | May be same DB tables initially; reporting module must not gain reject authority |

---

## 12. Observability of money movement

Minimum architectural expectation (details OPEN):

- Every reject and successful post can answer: **why**, **which edges funded**, **whether deficit was used**, **manual vs automatic**, **$0 reclaim**.
- Prefer structured domain events persisted as an **audit log** per budget.
- Application logs correlate with `budget_id`, command/request id, and period id (request id generation OPEN).
- Avoid unbounded production logging of full event bodies that contain balances or PII.

---

## 13. Explicit deferrals and non-goals

| Item | Status |
|------|--------|
| Authentication / authorization / IAM | **DEFERRED** — later issue; Security lead |
| Bank sync / real-world accounts | Product non-goal v1 |
| Microservices / Kafka | Not for v1 |
| Import-lint / ArchUnit module enforcement | Deferred tooling; conventions apply now |
| Multi-user sharing of one budget | Not in tenancy model |
| Cron / wall-clock auto-fund | Rejected as primary model |
| Full OpenAPI resource catalog | OPEN / later |
| Exact Angular state management library | OPEN / UI later |

---

## 14. Open questions

| # | Question | Why it matters |
|---|----------|----------------|
| O1 | Flyway migrate on `api` startup vs separate migrate init container? | Deploy ordering / fail behavior |
| O2 | ID strategy (UUID vs bigint) for Account/Budget/Node? | API and Angular models |
| O3 | REST resource naming and versioning (`/api/v1/...`)? | Client contract |
| O4 | Exact nginx vs Caddy; certificate issuance (Let’s Encrypt)? | Public host ops |
| O5 | Monorepo folder names / build tooling (Maven/Gradle, Angular CLI standalone)? | Repo scaffolding issue |
| O6 | ~~Period-open auto-fund transaction boundary~~ | **Resolved for v1:** same unit of work as `OpenPeriod` |
| O7 | Temporary public-API exposure policy before auth ships? | Security acceptance gate |
| O8 | Clock port testing strategy (fixed instant) for period boundaries? | QA |
| O9 | Whether reporting needs eventual separate read schema in v1 | Complexity control |

---

## 15. Acceptance criteria for this document (Issue #2)

Ready for owner acceptance when:

- [x] Decision summary §2 matches owner intent (modular monolith, Clean Architecture, stack, events, tenancy, Compose, auth deferral, convention-only modules)
- [x] Event-driven model does **not** rely on cron and does **not** break atomic funding commit
- [x] Angular has a dedicated section and a clear Compose serving approach
- [x] Auth is explicitly deferred without inventing IAM; public-host security baseline is stated
- [x] Module/package conventions are clear enough to scaffold the repo
- [ ] Owner promotes **Status** to **Accepted**

---

## 16. Decision log

| Date | Decision | Status | Notes |
|------|----------|--------|-------|
| 2026-10-16 | Issue #2 architecture consultation | Draft | Team options |
| 2026-10-16 | Modular monolith; Clean Architecture; Spring Boot + Angular + PostgreSQL + Flyway; event-driven (no cron primary); multi-account / one user per account / multi-budget; public Compose host; auth deferred; convention-only module enforcement | PROPOSED | Owner-locked; document written |
| 2026-10-16 | Period-open auto-fund in same UoW as `OpenPeriod`; every-X-periods on period lifecycle; single `web` SPA+TLS+proxy | PROPOSED | Owner-confirmed defaults |

---

## 17. Follow-on candidates (not created)

Draft ideas only — require human approval before opening GitHub issues:

1. Scaffold monorepo (Spring Boot + Angular + Flyway + Compose).
2. Security: authentication & authorization for public host (and temporary exposure policy O7).
3. ADR: concurrency / isolation level for funding commits.
4. ADR: audit event schema.
5. API surface design (OpenAPI).
6. Period open/close command UX + idempotent auto-fund application records.
