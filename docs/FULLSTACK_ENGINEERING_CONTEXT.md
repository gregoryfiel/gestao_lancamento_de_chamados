# Fullstack Engineering Context — Access Requests Portal

> **Audience:** any full-stack agent picking up this codebase for the first time.
> This document is the **single source of truth** for the complete stack —
> architecture, code standards, repository layout, HTTP contract, SPA patterns,
> operational rules and Git workflow.
>
> All technical artifacts (identifiers, comments, commit messages, JSON keys,
> OpenAPI) must be in **English**. UI strings rendered to the end-user follow
> pt-BR where the product specifies.

---

## 1. Product overview

The **Access Requests Portal** is a ticket-management tool for the BEES
Martech team at AB InBev. It manages access-request tickets (AD groups,
Unity Catalog roles, etc.) across their full lifecycle.

**Deployment target:** Databricks App (single process) — a **Python FastAPI
BFF** serving a **Vite + React SPA** plus Unity Catalog Delta tables as
persistence. Everything lives in the **ADO monorepo**
`data-platform-bees-consumer-mkt-strategy-insights` under
`apps/access-requests-portal/`.

**Canonical documentation repo:** GitHub
`gregoryfiel/gestao_lancamento_de_chamados` — docs live there; production
code lives in ADO. Never invert the two.

---

## 2. Repositories at a glance

| Repo | Location | Purpose |
|------|----------|---------|
| **ADO monorepo** | `https://dev.azure.com/ab-inbev/GHQ_B2B_Delta/_git/data-platform-bees-consumer-mkt-strategy-insights` | Production code, CI, Databricks deploy. All `apps/access-requests-portal/` edits happen here. |
| **GitHub docs** | `https://github.com/gregoryfiel/gestao_lancamento_de_chamados` | Canonical engineering docs (`docs/`), `.cursor/rules/`, legacy SPA history. |

**Local paths on the dev machine:**

```
ADO → C:\Users\gperuzzf\OneDrive - NTT DATA EMEAL\Documentos\ABI\GHQ_B2B_Delta\
      data-platform-bees-consumer-mkt-strategy-insights\

GitHub → C:\Users\gperuzzf\OneDrive - NTT DATA EMEAL\Documentos\ABI\Sandbox VSCode\
         gestao_lancamento_de_chamados\
```

---

## 3. Architecture

```
┌──────────────────────────── Databricks App (single process) ────────────────────────────┐
│                                                                                           │
│  Browser (SPA, Vite/React)  ──same-origin HTTPS──►  FastAPI BFF  ──►  Databricks SDK   │
│                                                         │                    │            │
│                                                   StaticFiles             SQL Warehouse   │
│                                                   /static/*                    │          │
│                                                                         Unity Catalog     │
│                                                                         Delta tables      │
│                                                                         (tickets +        │
│                                                                          ticket_events)   │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

Key rules that derive from this:

- **Same-origin**: the SPA calls `/api/v1/...` with no CORS, no API key in the browser.
- **Auth at the door**: Databricks Apps enforces SSO — the BFF never handles tokens; the browser never holds UC credentials.
- **BFF → UC**: BEES external SPN, credentials in Databricks Secrets. Never in `localStorage` or client-side code.
- **Audit trail**: every write (create, patch, soft-delete, bulk) records a row in `ticket_events` with `event_type`, `actor`, and a JSON `payload`.

---

## 4. Folder layout

```
apps/access-requests-portal/
│
├── app.py                          ← FastAPI entry point (create_app, mount router + static)
├── app.yaml                        ← Databricks App config (command, env, warehouse)
├── requirements.txt
│
├── api/
│   ├── routers/
│   │   └── tickets.py              ← HTTP only: parse → handler → map errors → status code
│   └── deps.py                     ← FastAPI Depends factories
│
├── application/
│   └── tickets/
│       ├── handlers.py             ← create_ticket, patch_ticket, delete_ticket, bulk_*
│       └── __init__.py
│
├── domain/
│   └── tickets/
│       ├── models.py               ← dataclasses / value objects / invariants
│       └── errors.py               ← RowVersionMismatchError, TicketNotFoundError, …
│
├── infrastructure/
│   └── databricks/
│       ├── ticket_repository_uc.py ← SQL Warehouse adapter (insert, update, soft_delete, list_events)
│       └── sql.py                  ← execute_statement wrapper, sql_string helper
│
├── ticket_store.py                 ← (legacy flat helpers used by handlers; migration in progress)
├── ticket_models.py                ← Pydantic request/response schemas (TicketRead, TicketCreate, …)
│
├── static/                         ← COPY of web/dist/* (served verbatim)
│   ├── index.html
│   ├── assets/                     ← hashed JS/CSS bundle
│   └── _smoke.html                 ← backend smoke console — NEVER EDIT FROM FE
│
└── web/                            ← SPA source
    ├── src/
    │   ├── App.tsx                 ← orchestrator: query state, repo, modals, drawers
    │   ├── main.tsx
    │   ├── index.css               ← Tailwind + custom overrides (light, print, animation)
    │   ├── contexts/
    │   │   ├── ThemeContext.tsx    ← dark/light toggle, localStorage, FOUC guard
    │   │   └── ToastContext.tsx    ← toast queue + ToastContainer
    │   ├── lib/
    │   │   ├── api-types.ts        ← re-exports from generated + SPA-only types
    │   │   ├── api-types.generated.ts  ← DO NOT EDIT — openapi-typescript output
    │   │   ├── repositories/
    │   │   │   ├── TicketsRepository.ts        ← port (interface)
    │   │   │   └── HttpTicketsRepository.ts    ← HTTP adapter (fetch)
    │   │   ├── useTickets.ts       ← query/loadMore/optimistic pending/create/patch/remove/bulk
    │   │   ├── useTicketEvents.ts  ← audit cursor pagination
    │   │   ├── pending-ticket.ts   ← PendingTicket type, builder, dedupe logic
    │   │   ├── uuid.ts             ← uuidv4 with crypto.randomUUID + Math.random fallback
    │   │   ├── event-diff.ts       ← parse audit payload → human-readable EventSummary
    │   │   ├── status-i18n.ts      ← status/priority pt-BR ↔ English labels
    │   │   ├── action-kind.ts      ← grant/revoke from extra field (workaround, see §12)
    │   │   ├── phase-tracker.ts    ← per-phase owner, SLA, traffic light
    │   │   ├── jira.ts             ← extractJiraKey from reference_url
    │   │   ├── ticket-age.ts       ← formatRelativeDate, isTerminalStatus
    │   │   ├── sort-tickets.ts     ← client-side tri-state sort
    │   │   └── tickets-query.ts    ← URL ↔ FilterQuery round-trip
    │   └── components/
    │       ├── layout/{Sidebar,Header}.tsx
    │       ├── ui/                 ← StatusBadge, Modal, Skeleton, ActionMenu,
    │       │                          AgeBadge, JiraBadge, ActionBadge, PhaseBadge,
    │       │                          ThemeToggle, Tooltip
    │       ├── TicketsTable.tsx    ← rows + sort headers + selection + pending rows
    │       ├── TicketDrawer.tsx    ← Details / Edit / Audit tabs
    │       ├── CreateTicketForm.tsx ← create + clone (initialValue)
    │       ├── ConfirmDeleteModal.tsx
    │       ├── FiltersBar.tsx      ← collapsible, chips when collapsed
    │       ├── BulkToolbar.tsx
    │       ├── Pagination.tsx      ← "Load more" + saving count
    │       ├── EventsTimeline.tsx  ← Audit tab with inline diff + collapsible raw JSON
    │       ├── PhaseTimeline.tsx   ← Details SLA breakdown
    │       ├── PrintReportHeader.tsx
    │       └── {EmptyState,ErrorBanner,HealthIndicator}.tsx
    ├── package.json
    ├── tsconfig.json
    ├── vite.config.ts              ← dev proxy: /api → http://127.0.0.1:8000
    └── README.md                   ← build instructions + manual smoke checklist
```

---

## 5. Backend standards

### 5.1 Mandatory quality bar

| Practice | What it means here |
|----------|--------------------|
| **PEP 8** | Style, naming, imports, line length, docstrings on public modules. |
| **SOLID** | Small types and functions; depend on abstractions (ports), not concrete SDK calls, inside domain/application layers. |
| **DDD** | Explicit **domain** (entities, invariants), **application** (use cases), **infrastructure** (Databricks SQL, HTTP, config). No business rules in route handlers. |
| **TDD** | Unit tests for domain/application with fake repos. Integration tests gated by `RUN_UC_INTEGRATION_TESTS=1`. A few route tests with `httpx.AsyncClient`. |

### 5.2 Hexagonal architecture (binding)

```
HTTP (FastAPI) ──► Application (handlers) ──► Domain (pure Python)
                           │
                      Ports (Protocol / ABC)
                           │
              Infrastructure adapters (Databricks SDK, SQL)
```

- **Domain + Application:** no `WorkspaceClient`, no raw SQL, no `fastapi.Request`.
- **Ports:** `TicketRepository`, `TicketEventRepository`, `Clock`, `IdGenerator`.
- **Driving adapter:** FastAPI routers translate HTTP ↔ Pydantic DTOs ↔ handler calls only.

### 5.3 Repository pattern

One repository per aggregate. Method names match use-case language:

```python
repo.get_by_id(ticket_id)
repo.list_active(filter_)
repo.insert(payload)
repo.update_with_version(ticket_id, patch, expected_version)
repo.soft_delete(ticket_id, row_version)
repo.list_events_for_ticket(ticket_id, before, limit)
```

### 5.4 Application services — one handler per use case

```python
# application/tickets/handlers.py
def create_ticket(repo: TicketRepository, payload: dict) -> dict: ...
def patch_ticket(repo: TicketRepository, ticket_id: str, patch: dict) -> dict: ...
def delete_ticket(repo: TicketRepository, ticket_id: str, row_version: int | None) -> dict: ...
```

New API behavior = new module + wire-up in `api/deps.py`, not growth of existing files.

### 5.5 Pydantic DTOs and OpenAPI

- HTTP boundary: Pydantic models (`TicketRead`, `TicketCreate`, `TicketUpdate`, …) in `ticket_models.py`.
- Versioned path: `/api/v1/...`
- Schema served at `/api/v1/openapi.json` — this is the SPA's contract.

### 5.6 Audit events (payload shapes)

Every write records a row in `ticket_events`. The `payload` JSON structure per `event_type`:

| `event_type` | `payload` shape |
|---|---|
| `created` | `{ "ticket": <TicketRead row> }` |
| `updated` | `{ "before": <row>, "after": <row>, "patch": <patch dict> }` |
| `deleted` | `{ "before": <row>, "row_version": <new int> }` |
| `bulk_create` | per row same as `created` |
| `bulk_update` | per row same as `updated` |

The SPA's `event-diff.ts` parses these to surface a human-readable diff in the Audit tab.

### 5.7 DDD layer dependency rules

| Layer | Allowed imports |
|-------|-----------------|
| **Domain** | stdlib, typing, datetime, domain-only helpers. |
| **Application** | domain, port interfaces. |
| **Infrastructure** | domain (if needed), `databricks-sdk`, SQL. |
| **API (FastAPI)** | application, Pydantic, FastAPI; infrastructure only via interfaces from `deps`. |

**Anti-pattern:** route handler building SQL strings or catching bare `Exception`.

### 5.8 Security

- Parameterized / escaped identifiers only (`sql_string()` helper).
- Never interpolate untrusted strings into SQL.
- Validate external IDs with explicit allow-lists (`assert_safe_ticket_id`).

### 5.9 TDD workflow

1. **Domain / application:** `pytest` with in-memory fake repos.
2. **Infrastructure:** opt-in integration tests (`RUN_UC_INTEGRATION_TESTS=1`).
3. **API:** route tests with `httpx.AsyncClient + ASGITransport` for status codes and shape.

---

## 6. HTTP contract (BFF ↔ SPA)

### 6.1 Endpoints

| # | Method | Path | Purpose |
|---|--------|------|---------|
| 1 | `GET` | `/api/v1/health` | Liveness |
| 2 | `GET` | `/api/v1/metrics` | Prometheus-style counters (ops/smoke only — **do not call from SPA**) |
| 3 | `GET` | `/api/v1/whoami` | BEES SPN smoke test |
| 4 | `GET` | `/api/v1/status` | UC table reachability + SQL probe |
| 5 | `GET` | `/api/v1/tickets` | List non-deleted tickets, filters + pagination |
| 6 | `POST` | `/api/v1/tickets` | Create + `created` audit event |
| 7 | `GET` | `/api/v1/tickets/{ticket_id}` | Fetch one ticket |
| 8 | `PATCH` | `/api/v1/tickets/{ticket_id}` | Partial update + `updated` event |
| 9 | `DELETE` | `/api/v1/tickets/{ticket_id}` | Soft-delete + `deleted` event |
| 10 | `POST` | `/api/v1/tickets:bulk-update` | Many partial updates; per-row outcome |
| 11 | `POST` | `/api/v1/tickets:bulk-create` | Many creates; per-row outcome |
| 12 | `GET` | `/api/v1/tickets/export` | Stream filtered XLSX (default) or CSV |
| 13 | `GET` | `/api/v1/tickets/{ticket_id}/events` | Audit rows (newest first), cursor |

### 6.2 Query parameters

**`GET /api/v1/tickets`**

| Name | Type | Default | Notes |
|------|------|---------|-------|
| `status` | string | — | exact match |
| `requester_email` | string | — | exact match |
| `opened_from` | `YYYY-MM-DD` | — | inclusive lower bound |
| `opened_to` | `YYYY-MM-DD` | — | inclusive upper bound; server validates `from <= to` |
| `limit` | int 1–500 | 200 | clamped server-side |
| `offset` | int ≥ 0 | 0 | offset pagination |

**`GET /api/v1/tickets/{id}/events`**

| Name | Type | Default | Notes |
|------|------|---------|-------|
| `before` | ISO timestamp | — | only events with `event_at < before` |
| `limit` | int 1–500 | 100 | clamped server-side |

**`DELETE /api/v1/tickets/{id}`**

| Name | Type | Default | Notes |
|------|------|---------|-------|
| `row_version` | int ≥ 1 | — | optional optimistic-locking precondition |

**`GET /api/v1/tickets/export`**

| Name | Type | Default | Notes |
|------|------|---------|-------|
| `format` | `xlsx` \| `csv` | `xlsx` | |
| `status`, `requester_email`, `opened_from`, `opened_to` | — | — | same as list |
| `columns` | comma-separated | whitelist | allowed: `ticket_id`, `status`, `priority`, `requester_email`, `employee_id`, `team`, `role`, `require_system`, `summary`, `opened_on`, `submitted_at`, `updated_at`, `jira_ticket` |
| `max_rows` | int 1–100000 | 100000 | `400` if more rows match |

### 6.3 Status codes

| Code | When |
|------|------|
| `200` | success on GET / PATCH / DELETE / bulk |
| `201` | success on `POST /tickets` (single create) |
| `400` | malformed body, invalid query, invalid enum value for `status`/`priority` |
| `404` | unknown or soft-deleted ticket |
| `409` | `row_version` mismatch |
| `503` | warehouse not reachable |

Bulk endpoints **never** return `409` at envelope level — each row carries its own `status_code`.

### 6.4 Enum validation (writes)

Writes (`POST`, `PATCH`, `bulk-update`, `bulk-create`) reject unknown `status`/`priority` with `400`.

**Allowed `status`:** `Aberto`, `Em Atendimento`, `Pendente`, `Resolvido`, `Cancelado` (plus English aliases).

**Allowed `priority`:** `low`, `baixa`, `normal`, `medium`, `media`, `high`, `alta`, `urgent`, `critical`, `critica`.

### 6.5 TypeScript models

```ts
interface TicketRead {
  ticket_id: string;
  requester_email: string;
  employee_id: string;
  require_system: string;
  role: string;
  team: string;
  reference_url: string;
  status: string;
  priority: string;
  opened_on: string;          // YYYY-MM-DD
  submitted_at: string;       // ISO 8601
  summary?: string | null;
  manager_email?: string | null;
  justification?: string | null;
  zone?: string | null;
  attachment_name?: string | null;
  extra?: string | null;
  created_at: string;
  updated_at: string;
  created_by_actor: string;   // "spn:<client-id>"
  updated_by_actor: string;
  row_version: number;
  is_deleted: boolean;
  [key: string]: unknown;     // extra="allow"
}

interface TicketEventRead {
  event_id: string;
  ticket_id: string;
  event_type: "created" | "updated" | "deleted" | string;
  event_at: string;
  actor: string;
  payload?: string | null;    // JSON string; see §5.6
}

interface TicketCreate { /* all fields except audit fields */ }
interface TicketUpdate { /* all fields optional + row_version */ }

interface BulkUpdateRequest  { items: { ticket_id: string; patch: TicketUpdate }[] }
interface BulkUpdateResponse { total: number; succeeded: number; failed: number; results: BulkUpdateResult[] }

interface BulkCreateRequest  { items: TicketCreate[] }
interface BulkCreateResponse { total: number; succeeded: number; failed: number; results: BulkCreateResult[] }
```

Authoritative source: `GET /api/v1/openapi.json`.

### 6.6 Cross-cutting

- **`X-Request-ID`:** every response includes it; echo in toast metadata.
- **Auth:** same-origin; Databricks Apps SSO at the door; no tokens in the SPA.
- **Optimistic locking:** send `row_version` on every write; on `409` refetch and show "stale row" banner without losing user input.
- **Pagination:** offset-based for tickets list; cursor-based (`before`) for events.

---

## 7. Frontend standards

### 7.1 Mandatory quality bar

| Practice | What it means here |
|----------|--------------------|
| **TypeScript strict** | `tsc -p tsconfig.json --noEmit` must pass with zero errors before every push. |
| **No `any` drift** | Write a small adapter type next to the consumer; never leak `any` across module boundaries. |
| **Hooks + repository pattern** | Components consume `useTickets` / `useTicketEvents`; data access goes through `HttpTicketsRepository`, never raw `fetch()` in components. |
| **Single source of truth** | Status/priority labels → `lib/status-i18n.ts`; phase SLA → `lib/phase-tracker.ts`; Jira key → `lib/jira.ts`. Do not duplicate. |
| **A11y first** | `aria-sort`, `role="menu"`, focus-visible rings, focus trapped inside modals/drawers. |
| **Print mode** | Every new UI element decides whether it appears in `@media print`. Use `print:hidden` on affordances by default. |

### 7.2 Tech stack

| Layer | Choice | Notes |
|-------|--------|-------|
| Build | **Vite 6** | Output → `web/dist/` → manually copied to `static/` |
| Framework | **React 19** | Functional components, hooks |
| Language | **TypeScript 5** | `strict: true` |
| Styling | **Tailwind CSS v4** | Custom overrides in `src/index.css` |
| Icons | **lucide-react** | Do not mix icon libraries |
| Type generation | **openapi-typescript** | `npm run generate:api` → `api-types.generated.ts` |

> **No `npm` on the agent machine.** Use Cursor's bundled Node binary:
>
> ```powershell
> $node = "C:\Program Files\cursor\resources\app\resources\helpers\node.exe"
> & $node node_modules\typescript\bin\tsc -p tsconfig.json --noEmit   # lint
> & $node node_modules\vite\bin\vite.js build                          # build
> & $node node_modules\vite\bin\vite.js                                # dev server
> ```

### 7.3 State and data flow

```
HttpTicketsRepository  ←── same-origin fetch (X-Request-ID, 409 mapping)
        ▲
   useTickets  (query, loadMore, optimistic pending, create/patch/remove/bulk)
   useTicketEvents  (cursor pagination)
        ▲
   App.tsx  (selected ticket, sort, filters, pending delete, toasts)
        ▲
   Components  (presentational; receive callbacks)
```

Standing patterns:

- **Query → URL.** `tickets-query.ts` round-trips filter state through `history.replaceState`.
- **Optimistic pending.** `useTickets.create` inserts a `PendingTicket` row immediately (via `pending-ticket.ts` helper). The modal closes in `< 200ms`. The POST runs in background. On success: replace pending with real row. On failure: mark as `_failed` → "Retry / Discard" inline.
- **Anti double-submit.** `Header` button goes into `Saving…` / disabled state while `creating === true`.
- **409 UX.** Any write returning `409` triggers `repo.get(ticket_id)`, shows "Row updated by another user" toast with the fresh data, keeps the form open.
- **Soft-delete is centralised.** `ConfirmDeleteModal` is mounted at App level.
- **Sort is client-side.** `sort-tickets.ts` tri-state cycle on the current page.

### 7.4 Visual identity

- **Dark-first.** Default `<html>` no class; `.light` toggles light theme. Persisted in `localStorage`. Inline script in `index.html` prevents FOUC.
- **Palette.** Slate-900 surfaces; indigo/cyan accents for primary actions; emerald/amber/rose for SLA traffic lights; rose for destructive.
- **Density.** `text-sm` body, `text-[11px]` metadata, `font-mono` for IDs/versions/Jira keys.
- **Phase ownership.** "Time open" is a per-phase traffic light:
  - **Aberto** → owner: **Martech**. SLA: ≤ 2h green / ≤ 8h yellow / > 8h red.
  - **Em Atendimento / Pendente** → owner: **BEES**. SLA: ≤ 24h / ≤ 72h / > 72h.
  - **Resolvido / Cancelado** → terminal, no SLA, no traffic light.
  - `rg -n '\bbis\b|BIS' apps/access-requests-portal/web/src` must return **zero** matches.
- **Print.** `@media print` hides sidebar, header chrome, drawer, modals, sort icons, three-dot menus. Table grows 100%; full cell borders; `PrintReportHeader` cover page. Optimistic rows hidden via `tr[data-optimistic="true"] { display: none }`.

### 7.5 Component conventions

- **Portal-based popovers.** `ActionMenu` uses `createPortal(menuNode, document.body)`. **Do not revert.** The table shell combines `overflow: hidden` + `overflow: auto`; the page header uses `backdrop-filter`; both trap `position: fixed`. Default `align='bottom-right'`; drawer footer opts into `align='top-right'`.
- **Modals.** `Modal.tsx` is the only primitive. Backdrop + Escape close; focus trapped; portal-mounted.
- **Status badges.** `StatusBadge` always goes through `statusLabel()` / `priorityLabel()` from `status-i18n.ts`.
- **Forms.** `CreateTicketForm` has `mode: 'create' | 'edit'`. In `create` mode it also accepts `initialValue` for the **clone** workflow.
- **Toasts.** `useToast()` from `ToastContext`. Types: `success`, `error`, `warning`, `info`. Auto-dismiss 5s; pinned errors until clicked.

### 7.6 Audit diff (EventsTimeline + event-diff.ts)

The Audit tab renders a human-readable diff above the collapsible raw JSON payload:

| event_type | What is shown |
|---|---|
| `created` | Note "Ticket created" + chip per non-empty field (`set` kind) |
| `updated` | Chip per changed field: `before → after` (amber) / `set` (emerald) / `cleared` (rose) |
| `deleted` | Note "Soft-deleted (was status: …)" |
| unknown | Nothing above the payload button |

17 tracked fields: `status`, `priority`, `requester_email`, `employee_id`, `manager_email`, `team`, `require_system`, `role`, `zone`, `reference_url`, `attachment_name`, `summary`, `justification`, `opened_on`, `submitted_at`, `is_deleted`, `extra`. Long fields truncated at 80 chars (full value in `title`).

---

## 8. Build, deploy & verification

### 8.1 Local development

```powershell
# Terminal 1 — BFF
cd apps\access-requests-portal
python -m uvicorn app:app --reload --host 127.0.0.1 --port 8000

# Terminal 2 — SPA (proxies /api to :8000 via vite.config.ts)
$node = "C:\Program Files\cursor\resources\app\resources\helpers\node.exe"
cd apps\access-requests-portal\web
& $node node_modules\vite\bin\vite.js
```

### 8.2 Production build

```powershell
$node = "C:\Program Files\cursor\resources\app\resources\helpers\node.exe"
$web  = ".\apps\access-requests-portal\web"
$static = ".\apps\access-requests-portal\static"

cd $web
& $node node_modules\typescript\bin\tsc -p tsconfig.json --noEmit
& $node node_modules\vite\bin\vite.js build

# Sync to static/ (BFF serves this folder verbatim)
# IMPORTANT: never touch _smoke.html
Remove-Item "$static\assets\index-*.js","$static\assets\index-*.css" -ErrorAction SilentlyContinue
Copy-Item "dist\index.html" "$static\index.html" -Force
Copy-Item "dist\assets\*"   "$static\assets\"    -Force
```

### 8.3 Verification checklist before every push

- [ ] `tsc --noEmit` clean.
- [ ] `vite build` clean — note new hashed asset names.
- [ ] `static/` synced (old hashes removed, new hashes added — both in the same commit).
- [ ] `_smoke.html` untouched: `git diff -- apps/access-requests-portal/static/_smoke.html` empty.
- [ ] No `bis` / `BIS`: `rg -n '\bbis\b|BIS' apps/access-requests-portal/web/src` → zero.
- [ ] No `console.log` left behind.
- [ ] No pt-BR strings in UI chrome (use `status-i18n.ts` for data values).

### 8.4 Deploy

A normal ADO PR merge triggers CI which rebuilds the Databricks App bundle.

---

## 9. Git workflow

### 9.1 ADO branches

| Branch | Owner | Purpose |
|--------|-------|---------|
| `master` | protected | production |
| `agent/access-requests-portal-backend` | BE agent | persistent BE work; reset to `origin/master` after each merge |
| `feat/access-requests-portal-frontend-<topic>` | FE agent / human | new capability; cut from `origin/master` |
| `fix/access-requests-portal-frontend-<topic>` | FE agent / human | bug fix or non-user-visible change |
| `feat/access-requests-portal-<topic>` | full-stack agent | cross-cutting feature |

**Hard rules:**
- Cut every branch from `origin/master`, never from a sibling branch.
- One logical unit per commit. `web/src` changes + `static/` rebuild in the **same** commit.
- Conventional Commits, English.
- **NEVER** open a PR automatically. Push the branch, hand the user a prefilled ADO URL.

### 9.2 GitHub branches

| Branch | Purpose |
|--------|---------|
| `main` | protected |
| `agent/backend-docs` | BE docs agent; reset to `origin/main` after merge |

### 9.3 Conventional Commit template

```
feat(access-requests-portal): <short description>

* web/src/<file>: <what / why>
* static/index.html + static/assets/*: rebuilt bundle.
```

### 9.4 Prefilled ADO PR URL

```
https://dev.azure.com/ab-inbev/GHQ_B2B_Delta/_git/
data-platform-bees-consumer-mkt-strategy-insights/
pullrequestcreate?sourceRef=<branch>&targetRef=master&title=<urlencoded-title>
```

### 9.5 Agent standing policy

| Repo | May commit + push | May open PR |
|------|-------------------|-------------|
| ADO — BE branch | yes | **NO — human opens** |
| ADO — FE / full-stack branches | yes | **NO — human opens** |
| GitHub — `agent/backend-docs` | yes | NO — human opens |

---

## 10. Deployment topology (Databricks Apps)

```
                   ┌──────────────── Databricks App ──────────────────┐
  Browser  ──SSO──►│  FastAPI (gunicorn / uvicorn)                     │
  (HTTPS)         │    StaticFiles("/")  →  static/                   │
                   │    APIRouter("/api/v1")                           │
                   │         │                                          │
                   │    Databricks SDK  ──►  SQL Warehouse             │
                   │                              │                    │
                   │                        Unity Catalog              │
                   │                        catalog.schema.tickets     │
                   │                        catalog.schema.ticket_events│
                   └──────────────────────────────────────────────────┘
```

**App size:** start with **Medium** (0.5 DBU/h). Scale to **Large** if you observe CPU/memory pressure.

**Business hours scheduling:** two Databricks Jobs:

```
app_start_morning  →  cron: 0 7 * * MON-FRI (America/Sao_Paulo)  →  apps.start(name=...)
app_stop_evening   →  cron: 0 20 * * MON-FRI                      →  apps.stop(name=...)
```

Stopped app: **no compute charge** (configuration retained).

---

## 11. SPA repository port (TicketsRepository)

```ts
interface TicketsRepository {
  list(query?: TicketsListQuery): Promise<TicketRead[]>;
  get(ticketId: string): Promise<TicketRead>;
  create(input: TicketCreate): Promise<TicketRead>;
  patch(ticketId: string, patch: TicketUpdate): Promise<TicketRead>;
  remove(ticketId: string, opts?: { rowVersion?: number }): Promise<TicketRead>;
  bulkUpdate(items: BulkUpdateRequest["items"]): Promise<BulkUpdateResponse>;
  bulkCreate(items: BulkCreateRequest["items"]): Promise<BulkCreateResponse>;
  export(opts: ExportQuery): Promise<{ blob: Blob; filename: string }>;
  listEvents(ticketId: string, opts?: { before?: string; limit?: number }): Promise<TicketEventRead[]>;
}
```

The `HttpTicketsRepository` implementation lives in `web/src/lib/repositories/HttpTicketsRepository.ts`. It is the **only** place that builds URLs, sets `credentials: "same-origin"`, maps `409` to `RepositoryRequestError`, and normalises `Content-Disposition` for binary responses.

---

## 12. Tactical workarounds (track for follow-up)

| What | Where | Why | Removal trigger |
|------|-------|-----|-----------------|
| `extra = "ACTION:REVOKE[;<note>]"` encodes grant/revoke action kind | `lib/action-kind.ts`, `CreateTicketForm`, `TicketsTable` left-border | Original schema had no `action_kind` column | BE branch `agent/access-requests-portal-backend` commit `49d2617` adds typed `action_kind`; once in SIT: swap FE to read/write the typed field, keep prefix as read-only fallback one release |
| `employee_id = "EXT:<company>"` encodes third-party requester | `CreateTicketForm` radio, `RequesterCell` | No typed `requester_kind` / `requester_company` columns | Same BE branch ships `requester_kind` + `requester_company`; same migration plan |
| Client-side sort | `lib/sort-tickets.ts` | BFF has no `order_by` | Add `order_by` to `GET /tickets` on BE; push sort server-side when list exceeds one page |
| FE-derived "Time open" / phase | `lib/phase-tracker.ts`, `PhaseBadge`, `PhaseTimeline` | Avoided new BE columns | If product wants exact per-phase SLAs in BI/exports, surface `phase_breakdown` from BE |

---

## 13. Known gotchas

- **`position: fixed` can be trapped.** Anything inside the table shell (`overflow: hidden` + `overflow: auto`) or the page header (`backdrop-filter`) needs a portal-rendered popover. `ActionMenu` already does this via `createPortal`. Copy that pattern for any new anchored UI.
- **Vite hashes shift CSS even when only TS changed.** Tailwind purges on `.tsx` edits. Always rebuild and re-sync `static/assets/` after any source change; never hand-edit hashed files.
- **OneDrive paths break `npm` heuristics.** Always use absolute paths when invoking Node or PowerShell. Quote paths with spaces; prefer `-LiteralPath`.
- **No `npm` on the agent machine.** See §7.2 — invoke tsc/vite directly via `node.exe`.
- **Commit message files must not contain a BOM** (especially on Windows PowerShell). Use `[System.IO.File]::WriteAllText($f, $msg, [System.Text.UTF8Encoding]::new($false))` before `git commit -F $f`, or use the `-m` heredoc pattern.
- **Sticky `<thead>` creates a stacking context** (`backdrop-blur`). Z-index ladder: `z-10` thead → `z-40` sidebar → `z-50` modals / portals.
- **Stale stashes can pollute Tailwind's class purge.** If `git status` shows files you did not touch, stash them with an explicit pathspec before building.
- **Concurrent branch edits cause checkout collisions.** If another shell or IDE is on a different branch in the same working tree, `git checkout` may fail. Use worktrees for parallel feature work.

---

## 14. Pending work and open follow-ups

| Item | Area | Status |
|------|------|--------|
| Swap `action_kind` FE to typed BE column | FE + BE | Blocked on BE `49d2617` being deployed to SIT |
| Swap `requester_kind` / `requester_company` | FE + BE | Same BE branch |
| Server-side `order_by` on `GET /tickets` | BE | Roadmap |
| `phase_breakdown` field from BE | BE | Roadmap (when BI/export needs exact SLAs) |
| Cursor pagination on tickets list | BE | Roadmap (when list > tens of thousands of rows) |
| `openapi-typescript` generation in CI | FE | Planned; currently manual |
| Vitest harness | FE | Not added yet; test manually per README checklist |

---

## 15. Onboarding checklist (first session)

1. Clone both repos (ADO = production, GitHub = docs).
2. Read **this document** end-to-end before writing any code.
3. Confirm current `master` is healthy: `git log --oneline -5` on ADO repo.
4. Run `tsc --noEmit` and `vite build` once — establish a clean baseline.
5. Start both BFF and SPA dev servers (§8.1); toggle dark/light theme; create + edit + delete a ticket; trigger a 409 by editing the same ticket in two tabs; check the per-phase traffic light.
6. Verify `rg '\bbis\b|BIS' apps/access-requests-portal/web/src` returns zero.
7. Open the Audit tab on a ticket with multiple events — confirm inline diff and collapsible JSON payload both render.
8. Pick up the next open item from §14 (consult the user for priority).

---

## 16. Responsibilities matrix

| Capability | BFF (Python) | SPA (React/TS) |
|------------|-------------|----------------|
| List / filter / paginate | SQL filters, offset/limit | FiltersBar, URL state, Load more, table rendering |
| Get one | fresh row + `row_version` | Drawer; calls `get` on 409 or open |
| Create (optimistic) | validate, persist, audit event | Modal closes < 200ms; pending row shown immediately; reconcile on response |
| Patch | optimistic lock via `row_version` | Edit form; sends `row_version`; maps 409 to stale-row banner |
| Delete (soft) | `is_deleted`, audit event | ConfirmDeleteModal; pass `row_version` |
| Bulk update | per-row outcomes, no batch abort | Multi-select toolbar, per-row badges |
| Bulk create / import | per-row 201/400 | File picker, CSV/XLSX parse, preview, call `bulkCreate` |
| Audit events | cursor `before` | Timeline in Audit tab: inline diff + collapsible raw JSON |
| Export (XLSX/CSV) | filtered bytes, `Content-Disposition` | Export modal → blob download via `URL.createObjectURL` |
| PDF for stakeholders | **not provided** (by design) | Print stylesheet + `window.print()` |
| OpenAPI / types | serves `/api/v1/openapi.json` | Generates TS types; implements `HttpTicketsRepository` |
| Auth / secrets | BEES SPN, UC grants, warehouse ID | Same-origin fetch only; no tokens or blobs in `localStorage` |
| Metrics | `GET /api/v1/metrics` (ops/smoke) | **Do not call from the SPA** |
