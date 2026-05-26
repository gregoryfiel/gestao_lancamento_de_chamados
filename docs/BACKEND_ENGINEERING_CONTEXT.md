# Backend engineering context — BFF, Databricks Apps, and API evolution

> **Single source of truth** for Python back-end work aligned with this product
> (FastAPI BFF, Databricks App bundles, Unity Catalog persistence). **English**
> technical standard. Use this doc when implementing or reviewing server-side
> code and when evolving contracts consumed by the SPA.

**Related:** [`BACKEND_EXCEL_HANDOFF.md`](./BACKEND_EXCEL_HANDOFF.md) (product
intent, contract-first), [`DATABRICKS_APPS_TARGET_ARCHITECTURE.md`](./DATABRICKS_APPS_TARGET_ARCHITECTURE.md)
(deploy topology, cost, scheduling).

---

## 1. Mandatory quality bar (from now on)

| Practice | What it means here |
|----------|-------------------|
| **PEP 8** | Style, naming, imports, line length, docstrings on public modules. Prefer `ruff check` / `ruff format` or `black` in CI when the repo adds a gate. |
| **SOLID** | Small types and functions; depend on abstractions (ports), not concrete SDK calls, inside domain/application layers; one reason to change per module. |
| **DDD** | Explicit **domain** (entities, invariants), **application** (use cases, orchestration), **infrastructure** (Databricks SQL, HTTP, config). No business rules hidden in route handlers. |
| **TDD** | Red–green–refactor for non-trivial behavior: unit tests for domain and use cases; integration tests behind flags for warehouse/UC (see §6). |

Non-goals: over-abstraction (no “framework inside a framework”), speculative patterns for features that do not exist yet.

---

## 2. Design direction for a pluggable back end the front end can reuse

The SPA will call **same-origin** `/api/v1/...` and should depend on a **stable
contract**, not on implementation details. Favor patterns that:

- isolate **Databricks / SQL** behind interfaces;
- keep **FastAPI routes** thin (HTTP mapping only);
- let you **add endpoints** by adding use cases + wiring, not by growing god-files.

### 2.1 Primary pattern: Hexagonal architecture (Ports and Adapters)

- **Domain + application (hexagon center):** pure Python rules (validation,
  optimistic locking semantics, id formats). No `WorkspaceClient`, no
  `fastapi.Request`, no raw SQL strings.
- **Ports:** `Protocol` or abstract base classes, e.g. `TicketRepository`,
  `TicketEventRepository`, `Clock`, `IdGenerator`.
- **Adapters (infrastructure):** UC + SQL warehouse implementation of those
  ports; workspace auth; environment-backed configuration.
- **Driving adapter:** FastAPI routers translate HTTP ↔ DTOs ↔ use case calls.

**Why:** swapping SIT vs PROD warehouses, mocking in tests, or later adding a
read replica / different store does not force changes in domain code.

### 2.2 Repository pattern (persistence port)

- One repository (or split read/write interfaces if needed) per aggregate root
  you persist (`Ticket`, `TicketEvent` stream).
- Methods reflect **use-case language**: `get_by_id`, `list_active`,
  `insert`, `update_with_version`, `list_events_for_ticket` — not generic
  `execute_query` scattered in routes.

**Why:** the future **HTTP repository on the SPA** (see handoff doc) mirrors the
same operations; naming alignment reduces drift.

### 2.3 Application services / use cases (facade per operation)

- One module or class per vertical slice: e.g. `CreateTicketHandler`,
  `PatchTicketHandler`, `ListTicketEventsHandler`.
- Each exposes a small `execute(command)` / `run(dto)` entry point; returns
  domain result or raises domain exceptions mapped to HTTP in the router layer.

**Why:** new API methods = new use case + wire-up, not conditional spaghetti in
a single 800-line module.

### 2.4 DTOs and OpenAPI as the FE contract

- **Pydantic** models at the HTTP boundary (`request` / `response` schemas) stay
  separate from **domain** objects if shapes differ.
- Publish **`/api/v1/openapi.json`**; keep versions in the path (`/api/v1`).
  Optional later: generate a TypeScript client for the SPA from the same spec.

**Why:** front end and back end share one machine-readable contract; reduces
field-name drift with [`src/types.ts`](../src/types.ts) during migration.

### 2.5 Dependency injection (FastAPI `Depends`)

- Construct adapters once per request (or use scoped providers); inject
  repositories/handlers into routers.

**Why:** testability and explicit wiring graph; avoids global singletons except
for truly immutable config.

### 2.6 Supporting patterns (use when a concrete need appears)

| Pattern | When |
|---------|------|
| **Strategy** | Multiple auth or storage backends (e.g. dev file vs UC). |
| **Factory** | Build `WorkspaceClient` / warehouse ID from env without `import` side effects everywhere. |
| **Unit of Work** | If you need transactional multi-table updates beyond current single-flow SQL. |
| **Outbox** | If you must publish to Kafka/Jobs after commit (not required for MVP). |

---

## 3. Suggested package layout (evolutionary, not a big-bang rewrite)

Use incremental moves from a flat `app.py` layout toward:

```
app/  # or access-requests-portal/
  main.py                 # create_app(), mount routers, static
  api/
    routers/
      tickets.py          # HTTP only: parse, call handler, map errors → status
    deps.py               # FastAPI Depends factories
  application/
    tickets/
      create_ticket.py
      patch_ticket.py
      list_events.py
  domain/
    tickets/
      models.py           # dataclasses / value objects, invariants
      errors.py           # domain-specific exceptions
  infrastructure/
    databricks/
      ticket_repository_uc.py
      sql.py              # execute_statement wrapper, typing helpers
  contracts/              # optional: OpenAPI extras, shared JSON schemas
```

New **methods and classes** should land in the layer that matches their
responsibility; resist adding logic to `api/` beyond mapping.

---

## 4. DDD boundaries (practical)

| Layer | Allowed dependencies |
|-------|----------------------|
| **Domain** | stdlib, typing, `datetime`, domain-only helpers. |
| **Application** | domain, port interfaces. |
| **Infrastructure** | domain (if needed), SDKs (`databricks-sdk`), SQL. |
| **API (FastAPI)** | application, Pydantic, FastAPI, infrastructure only via interfaces wired in `deps`. |

**Anti-pattern:** route handler building SQL strings or catching broad
`Exception` and returning opaque 500s without logging context (request id, use
case name).

---

## 5. TDD workflow

1. **Domain / application:** `pytest` unit tests with fake repositories (in-memory
   dicts or `Protocol` fakes).
2. **Infrastructure:** opt-in integration tests gated by env
   (`RUN_UC_INTEGRATION_TESTS=1`) so CI without credentials stays green.
3. **API:** a few route tests (e.g. `httpx.AsyncClient` + `ASGITransport`) for
   status codes and response shape, not full warehouse behavior.

When the harness does not exist yet, document manual validation in the PR and
add the smallest test module that locks the next behavior.

---

## 6. Operational extras (already aligned with product)

- **Request correlation:** propagate `X-Request-ID` through logs and responses
  where the platform allows it.
- **Security:** parameterized / escaped identifiers only; never interpolate
  untrusted strings into SQL; validate external ids with explicit allow-lists.

---

## 7. Summary

**Default stack for this product’s back end:** **Hexagonal + Repository +
Application use cases + Pydantic DTOs + FastAPI DI**, with **OpenAPI** as the
shared contract with the SPA. Enforce **PEP 8, SOLID, DDD layering, and TDD**
for new and touched code so new endpoints remain **plug-in friendly** and the
front end can evolve against a **stable, documented API surface**.
