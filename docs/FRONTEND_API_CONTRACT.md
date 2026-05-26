# Front-end API contract — Access Requests Portal BFF

> **Audience:** the front-end agent for `gestao_lancamento_de_chamados`. This
> document is the **single source of truth** for the HTTP contract exposed by
> the BFF (Databricks App `access-requests-portal`) and the migration plan
> from `localStorage` to a real back end.
>
> **Companion docs:**
> - [`BACKEND_ENGINEERING_CONTEXT.md`](./BACKEND_ENGINEERING_CONTEXT.md) — server-side standards.
> - [`DATABRICKS_APPS_TARGET_ARCHITECTURE.md`](./DATABRICKS_APPS_TARGET_ARCHITECTURE.md) — deploy topology.
> - [`BACKEND_EXCEL_HANDOFF.md`](./BACKEND_EXCEL_HANDOFF.md) — product intent.

---

## 1. TL;DR for the front-end agent

- The BFF and the SPA are the **same Databricks App** → call **same-origin**
  (`/api/v1/...`), no CORS, no API key in the browser.
- All technical artifacts (identifiers, comments, JSON keys, OpenAPI) are
  **English snake_case**. UI strings stay **pt-BR** per the frontend rule.
- The OpenAPI schema is served at **`/api/v1/openapi.json`**. Generate the TS
  client from it; do **not** hand-roll types.
- Every response carries an **`X-Request-ID`** header for log correlation.
- Optimistic locking is server-authoritative via **`row_version`** — surface
  `409` to the user as “registo desatualizado, recarregue”.
- The current SPA `Ticket` interface has **pt-BR field names** but the API
  uses **snake_case English**. Section 6 has the mapping table; build a
  translator in the repository adapter, not in components.

---

## 2. Endpoints (current contract)

| # | Method | Path | Purpose |
|---|--------|------|---------|
| 1 | `GET` | `/api/v1/health` | Liveness. |
| 2 | `GET` | `/api/v1/whoami` | BEES SPN smoke test (server-only auth). |
| 3 | `GET` | `/api/v1/status` | UC table reachability + SQL probe. |
| 4 | `GET` | `/api/v1/tickets` | List non-deleted tickets, filters + pagination. |
| 5 | `POST` | `/api/v1/tickets` | Create ticket + `created` audit event. |
| 6 | `GET` | `/api/v1/tickets/{ticket_id}` | Fetch one ticket. |
| 7 | `PATCH` | `/api/v1/tickets/{ticket_id}` | Partial update + `updated` event. |
| 8 | `DELETE` | `/api/v1/tickets/{ticket_id}` | Soft delete (`is_deleted=true`) + `deleted` event. |
| 9 | `POST` | `/api/v1/tickets:bulk-update` | Many partial updates in one round trip; per-row outcome. |
| 10 | `GET` | `/api/v1/tickets/{ticket_id}/events` | Audit rows (newest first), cursor pagination. |

### 2.1 Query parameters

**`GET /api/v1/tickets`**

| Name | Type | Default | Notes |
|------|------|---------|-------|
| `status` | string | — | exact match on the `status` column. |
| `requester_email` | string | — | exact match on the `requester_email` column. |
| `opened_from` | `YYYY-MM-DD` | — | inclusive lower bound on `opened_on`. |
| `opened_to` | `YYYY-MM-DD` | — | inclusive upper bound on `opened_on`. Server validates `from <= to`. |
| `limit` | int 1..500 | 200 | clamped server-side. |
| `offset` | int >= 0 | 0 | offset pagination (no cursor today). |

**`GET /api/v1/tickets/{ticket_id}/events`**

| Name | Type | Default | Notes |
|------|------|---------|-------|
| `before` | ISO timestamp | — | only events with `event_at < before` (cursor). |
| `limit` | int 1..500 | 100 | clamped server-side. |

**`DELETE /api/v1/tickets/{ticket_id}`**

| Name | Type | Default | Notes |
|------|------|---------|-------|
| `row_version` | int >= 1 | — | optional optimistic-locking precondition. |

### 2.2 Status codes

| Code | When |
|------|------|
| `200` | success on `GET`, `PATCH`, `DELETE`, `POST :bulk-update`. |
| `201` | success on `POST /tickets` (create). |
| `400` | malformed body, invalid query (`opened_from > opened_to`), invalid `ticket_id` format. |
| `404` | unknown / soft-deleted ticket. |
| `409` | `row_version` mismatch (optimistic-lock violation). |
| `503` | downstream warehouse not reachable. |

Bulk update **never** returns `409` at the envelope level: each row carries
its own `status_code` (200 / 400 / 404 / 409) inside `results[]`.

---

## 3. Response models (TypeScript-shaped)

> Authoritative source: `/api/v1/openapi.json`. The shapes below are pasted
> for review; regenerate the TS client whenever this doc changes.

```ts
export interface TicketRead {
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
  created_at: string;         // ISO 8601
  updated_at: string;         // ISO 8601
  created_by_actor: string;   // e.g. "spn:<client-id>"
  updated_by_actor: string;
  row_version: number;
  is_deleted: boolean;
  // Forward-compatible extra keys allowed (model_config extra="allow").
  [key: string]: unknown;
}

export interface TicketEventRead {
  event_id: string;
  ticket_id: string;
  event_type: "created" | "updated" | "deleted" | string;
  event_at: string;           // ISO 8601
  event_date?: string | null; // YYYY-MM-DD (audit partition)
  actor: string;
  payload?: string | null;    // JSON string, opaque to UI today
  [key: string]: unknown;
}

export interface BulkUpdateRequest {
  items: Array<{
    ticket_id: string;
    patch: TicketUpdate;
  }>; // 1..200 entries
}

export interface BulkUpdateResult {
  ticket_id: string;
  ok: boolean;
  status_code: 200 | 400 | 404 | 409;
  row?: TicketRead | null;
  error?: string | null;
}

export interface BulkUpdateResponse {
  total: number;
  succeeded: number;
  failed: number;
  results: BulkUpdateResult[];
}

export interface TicketCreate {
  requester_email: string;
  employee_id: string;
  require_system: string;
  role: string;
  team: string;
  reference_url?: string;     // defaults to ""
  status: string;
  priority: string;
  opened_on: string;          // YYYY-MM-DD
  submitted_at: string;       // ISO 8601
  summary?: string;
  manager_email?: string;
  justification?: string;
  zone?: string;
  attachment_name?: string;
  extra?: string;
}

export interface TicketUpdate {
  // All fields optional; omit those that you do not change.
  requester_email?: string;
  employee_id?: string;
  require_system?: string;
  role?: string;
  team?: string;
  reference_url?: string;
  status?: string;
  priority?: string;
  opened_on?: string;
  submitted_at?: string;
  summary?: string;
  manager_email?: string;
  justification?: string;
  zone?: string;
  attachment_name?: string;
  extra?: string;
  is_deleted?: boolean;
  row_version?: number;       // when set, server enforces optimistic locking
}

export interface ApiError {
  detail: string; // FastAPI default error envelope
}
```

### 3.1 Generating the typed client

Add a dev-only step on the SPA repo (planned, not yet wired):

```bash
npx openapi-typescript "$BFF_BASE/api/v1/openapi.json" -o src/lib/api-types.ts
```

`BFF_BASE` is empty in production (same-origin); for local dev hit the
deployed App URL and copy `api-types.ts` into the repo.

---

## 4. Cross-cutting concerns

### 4.1 Auth

- **Browser → BFF:** same-origin, no token in the SPA; the Databricks App
  enforces SSO at the front door.
- **BFF → Unity Catalog:** BEES external SPN, server-only. The browser must
  **never** hold UC credentials. Retire the simulated SharePoint Graph keys
  in `SheetDatabase.tsx` as part of the migration.

### 4.2 `X-Request-ID`

Every BFF response includes `X-Request-ID`. The SPA should:

1. Echo the value into UI logs / toast metadata for support.
2. Optionally **send** an `X-Request-ID` header on outgoing fetches when the
   user reports an issue (the BFF will reuse the one provided).

### 4.3 Optimistic locking (`row_version`)

- Read `row_version` from the latest `TicketRead` you have on screen.
- On `PATCH` / `DELETE` / bulk patch, send the same `row_version`.
- On `409`, refetch the row, show a non-destructive “registo desatualizado,
  carregue para recarregar” message, do **not** silently overwrite.

### 4.4 Pagination

- **Tickets list:** offset-based (`limit` + `offset`) for now. A cursor
  variant (`submitted_at` + tiebreaker on `ticket_id`) is on the roadmap if
  the dataset grows past tens of thousands of rows.
- **Events:** cursor-based via `before=<event_at>`; safe for long histories.

### 4.5 Errors

FastAPI default envelope: `{ "detail": "..." }`. The bulk endpoint never
returns `409` at the envelope level; map per-row `status_code` to a per-row
badge in the UI.

---

## 5. Repository pattern in the SPA

`docs/BACKEND_EXCEL_HANDOFF.md` §3.3 already proposed a thin repository
seam. With the BFF in place, the contract is:

```ts
export interface TicketsRepository {
  list(query?: TicketsListQuery): Promise<TicketRead[]>;
  get(ticketId: string): Promise<TicketRead>;
  create(input: TicketCreate): Promise<TicketRead>;
  patch(ticketId: string, patch: TicketUpdate): Promise<TicketRead>;
  remove(ticketId: string, opts?: { rowVersion?: number }): Promise<TicketRead>;
  bulkUpdate(items: BulkUpdateRequest["items"]): Promise<BulkUpdateResponse>;
  listEvents(ticketId: string, opts?: { before?: string; limit?: number }): Promise<TicketEventRead[]>;
}

export interface TicketsListQuery {
  status?: string;
  requesterEmail?: string;
  openedFrom?: string;        // YYYY-MM-DD
  openedTo?: string;          // YYYY-MM-DD
  limit?: number;
  offset?: number;
}
```

Two implementations should coexist behind a feature flag during cutover:

| Adapter | Source of truth | When |
|---------|-----------------|------|
| `LocalStorageTicketsRepository` | `localStorage` key `ab_inbev_tickets_db_v8`. | today / offline fallback. |
| `HttpTicketsRepository` | the BFF described here. | once cutover is approved. |

The HTTP adapter is also the place that **translates field names** (see §6),
so React components keep using the existing pt-BR `Ticket` interface or its
modernized successor without churn.

### 5.1 Suggested skeleton

```ts
// src/lib/repositories/HttpTicketsRepository.ts
export class HttpTicketsRepository implements TicketsRepository {
  constructor(private readonly base: string = "/api/v1") {}

  private async request<T>(input: RequestInfo, init?: RequestInit): Promise<T> {
    const res = await fetch(input, { credentials: "same-origin", ...init });
    if (!res.ok) {
      const err = (await res.json().catch(() => ({}))) as ApiError;
      throw new Error(err.detail ?? `${res.status} ${res.statusText}`);
    }
    return res.json() as Promise<T>;
  }

  list(q: TicketsListQuery = {}): Promise<TicketRead[]> {
    const qs = new URLSearchParams();
    if (q.status) qs.set("status", q.status);
    if (q.requesterEmail) qs.set("requester_email", q.requesterEmail);
    if (q.openedFrom) qs.set("opened_from", q.openedFrom);
    if (q.openedTo) qs.set("opened_to", q.openedTo);
    if (q.limit !== undefined) qs.set("limit", String(q.limit));
    if (q.offset !== undefined) qs.set("offset", String(q.offset));
    return this.request(`${this.base}/tickets?${qs}`);
  }
  // ...remaining methods omitted for brevity.
}
```

---

## 6. Field mapping (current SPA `Ticket` ↔ API `TicketRead`)

The SPA today uses pt-BR field names; the API uses English snake_case. **Do
not rename API fields.** Translate inside the HTTP adapter.

| SPA `Ticket` (`src/types.ts`) | API `TicketRead` |
|-------------------------------|------------------|
| `id` | `ticket_id` |
| `pessoaEmail` | `requester_email` |
| `employeeId` | `employee_id` |
| `requireSystem` | `require_system` |
| `role` | `role` |
| `equipe` | `team` |
| `url` | `reference_url` |
| `status` | `status` |
| `prioridade` | `priority` |
| `dataAbertura` | `opened_on` (date-only) |
| `dataInclusao` | `submitted_at` |
| `resumo` | `summary` |
| `managerEmail` | `manager_email` |
| `justificativa` | `justification` |
| `zona` | `zone` |
| `anexoNome` | `attachment_name` |

API-only fields (no pt-BR alias): `created_at`, `updated_at`,
`created_by_actor`, `updated_by_actor`, `row_version`, `is_deleted`,
`extra`. Surface these as needed (audit panel, optimistic locking, etc.).

`TicketStatus` and `TicketPriority` enums on the SPA are **not** enforced
by the API today — the column accepts arbitrary strings. The SPA should
keep using its enums as the canonical user-facing list and let the BFF
store whatever string you send.

---

## 7. Migration plan from `localStorage` to API

Goal: cut the SPA over without losing tickets users already created locally.

1. **Land the repository seam (current SPA, no BFF call yet).**
   - Introduce `TicketsRepository` interface and refactor consumers
     (`SheetDatabase.tsx`, `Dashboard.tsx`, forms) to talk to the
     repository, not directly to `localStorage`.
2. **Add the HTTP adapter behind a flag.**
   - `useBackend = false` by default.
   - Build `HttpTicketsRepository` with the field mapping.
3. **Dual-read week.**
   - When `useBackend = true`, the SPA reads from the API but keeps
     writing to `localStorage` as a backup.
4. **One-shot import.**
   - Use `POST /api/v1/tickets:bulk-update` *not* for import (it is
     update-only); call `POST /api/v1/tickets` per row, or land a
     dedicated bulk-create endpoint first if list is large.
5. **Cutover.**
   - Flip `useBackend = true` permanently, stop writing to
     `localStorage`, retire simulated Graph keys.
6. **Versioning.**
   - `localStorage` key `ab_inbev_tickets_db_v8` should be **bumped**
     (`_v9`) when the SPA payload shape changes during the cutover, to
     avoid hydrating stale rows from a different schema.

---

## 8. Sample wire payloads

### 8.1 `POST /api/v1/tickets` (create)

**Request**

```json
{
  "requester_email": "alice@ab-inbev.com",
  "employee_id": "EMP-1",
  "require_system": "AD Group - Unity Catalog",
  "role": "AADS_A_BEES_UC_CONSUMER_MARKETING_ENGINEER",
  "team": "Martech",
  "reference_url": "",
  "status": "Aberto",
  "priority": "normal",
  "opened_on": "2026-05-26",
  "submitted_at": "2026-05-26T19:47:18Z",
  "summary": "Acesso ao catálogo silver"
}
```

**Response 201 — `TicketRead`** (truncated): see §3.

### 8.2 `POST /api/v1/tickets:bulk-update` (partial failure)

**Request**

```json
{
  "items": [
    { "ticket_id": "abc-1", "patch": { "status": "Em Atendimento", "row_version": 1 } },
    { "ticket_id": "abc-2", "patch": { "status": "Em Atendimento", "row_version": 1 } },
    { "ticket_id": "missing", "patch": { "status": "Em Atendimento" } }
  ]
}
```

**Response 200 — `BulkUpdateResponse`**

```json
{
  "total": 3,
  "succeeded": 1,
  "failed": 2,
  "results": [
    { "ticket_id": "abc-1", "ok": true, "status_code": 200, "row": { "...": "..." } },
    { "ticket_id": "abc-2", "ok": false, "status_code": 409, "error": "row_version mismatch" },
    { "ticket_id": "missing", "ok": false, "status_code": 404, "error": "ticket not found" }
  ]
}
```

---

## 9. Out of scope (today)

The following items are **not** offered by the BFF yet; if the UI needs
them, open a coordination issue / branch:

- User-level identity in requests (the BFF only authenticates against UC
  with the BEES SPN; user identity is established by the Databricks Apps
  proxy and is not surfaced in `/api/v1/whoami` as the **caller**).
- CSV export endpoint (`GET /api/v1/export.csv`).
- Bulk **create**.
- Server-side full-text search across `summary` / `justification`.

---

## 10. Quick checklist for the front-end agent

- [ ] Read this doc + `BACKEND_ENGINEERING_CONTEXT.md` once.
- [ ] Generate `src/lib/api-types.ts` from `/api/v1/openapi.json`.
- [ ] Add `TicketsRepository` interface and refactor consumers to use it.
- [ ] Implement `HttpTicketsRepository` with the field mapping table (§6).
- [ ] Wire a `useBackend` flag (env or runtime toggle).
- [ ] Surface `409` as an inline “registo desatualizado” notice with a refresh
      action, not a destructive overwrite.
- [ ] Render bulk update outcomes per row (badge per `results[].status_code`).
- [ ] Pass `X-Request-ID` through to the toast system for support reports.
- [ ] Bump the `localStorage` key when payload shape changes.

---

**Last updated:** 2026-05-26 (matches BFF state on ADO branch
`feat/access-requests-portal-bulk-update-and-response-models`).
