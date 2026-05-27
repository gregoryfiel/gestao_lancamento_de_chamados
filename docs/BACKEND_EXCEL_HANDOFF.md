# Backend / Excel integration — handoff and intent

> **pt-BR (contexto):** Este arquivo registra a decisão de **priorizar backend e integração com Excel** antes de grandes ondas de modernização do front. O texto técnico abaixo está em **inglês** (padrão do projeto). Use-o como briefing para o agente ou desenvolvedor de backend.

**Engineering standards (Python BFF, Databricks, plugável):** see
[`BACKEND_ENGINEERING_CONTEXT.md`](./BACKEND_ENGINEERING_CONTEXT.md) (PEP 8,
SOLID, DDD, TDD, hexagonal + repository + OpenAPI contract with the SPA).

**Front-end consumer contract** (endpoints, types, optimistic locking, field
mapping `pessoaEmail ↔ requester_email` etc., and the `localStorage → API`
migration plan): [`FRONTEND_API_CONTRACT.md`](./FRONTEND_API_CONTRACT.md).

> **Status (2026-05-27):** Excel-as-ledger questions in §5 below are largely
> **superseded** by Unity Catalog + `GET /tickets/export`. For export columns,
> bulk import, and **BFF vs SPA responsibilities**, use
> [`FRONTEND_API_CONTRACT.md`](./FRONTEND_API_CONTRACT.md) **§11**.

---

## 1. Current state (frontend-only)

- The app has **no server-side persistence**. Ticket data is stored in the browser under `localStorage` key `ab_inbev_tickets_db_v8` (see [`src/App.tsx`](../src/App.tsx)).
- [`src/components/SheetDatabase.tsx`](../src/components/SheetDatabase.tsx) is a large client module (~2.4k lines) that includes **filters, CSV/TSV import/export, bulk actions**, and **simulated** integrations (not production-grade).
- **Excel / SharePoint** are not authoritative today; they are approximated in the UI. Any "database" behavior is local until a real backend exists.

---

## 2. Decision (product owner intent)

**Treat backend + Excel (or cloud workbook) as the first priority.** After that layer is defined and minimally working, return to the frontend modernization waves (design tokens, dashboard honesty, form deduplication, sheet modularization).

Rationale:

- Large refactors of the sheet layer **risk being redone** once the real read/write contract and storage model exist.
- Multi-user, audit, and compliance needs **cannot** be satisfied by `localStorage` alone.

---

## 3. What we aligned on (effectiveness strategy)

### 3.1 Do not block the frontend forever

- The frontend can still evolve **cosmetically** (shell, typography, chart styling) in small PRs if needed.
- **Avoid** deep modularization of `SheetDatabase.tsx` and **avoid** persisting more simulated credentials until the backend owns auth and storage.

### 3.2 Contract-first

1. Choose the **source of truth**: cloud Excel (SharePoint / OneDrive) via Microsoft Graph, a dedicated API + database with Excel as export only, or a hybrid (API caches rows, Excel is edited by ops).
2. Publish a minimal contract: **list tickets**, **create/update ticket**, optional **bulk update**, and **import/export** semantics if Excel remains the ledger.
3. Keep the JSON shape close to the existing [`Ticket`](../src/types.ts) type unless there is a strong reason to rename fields; any schema change must ship a **migration plan** and a versioned storage strategy when the client still uses `localStorage` during transition.

### 3.3 Repository boundary (recommended for the eventual front swap)

- Introduce a thin persistence interface on the frontend (e.g. `TicketsRepository`: `list`, `save`, `delete`, `clear`, `import`) with:
  - `LocalStorageTicketsRepository` today.
  - `HttpTicketsRepository` (or Graph-backed adapter) when the backend is ready.
- This allows **one PR** to swap persistence without rewriting forms and the sheet UI.

### 3.4 Security

- **Never** treat simulated Graph tokens in `localStorage` as production-ready. Backend should own OAuth / secrets. Frontend should only receive short-lived tokens or call your API, not store raw secrets.

---

## 4. Suggested backend scope (phased)

### Phase A — Foundations

- Authentication model (who can read/write the workbook or API).
- Where the workbook lives (site, drive, file id) and **environment configuration** (no secrets in the repo).

### Phase B — Minimal API surface

- `GET /tickets` (filters optional in v2).
- `POST /tickets` and `PATCH /tickets/:id` (or batch endpoint for bulk status updates).
- Optional: `POST /import` / `GET /export` if CSV remains a bridge.

### Phase C — Excel / Graph specifics (if Excel is the ledger)

- Map each `Ticket` field to **columns** (stable header row or named table).
- Define **concurrency**: last-write-wins, ETag / `if-match`, or row version column.
- Define **audit**: who changed what (may require a separate audit table if Excel alone is insufficient).

### Phase D — Hardening

- Rate limits, pagination, idempotency keys for bulk operations.
- Observability (structured logs, correlation ids).

---

## 5. Open questions for the backend owner (answer before coding deep)

1. **Is Excel the system of record**, or is it a **view/export** of a real database?
2. **Single shared workbook** vs **one file per team** vs **database + export**?
3. **Expected concurrency** (how many editors at once)?
4. **Offline / latency** requirements (does the UI need optimistic updates and retry queues)?
5. **PII policy** for emails and employee ids in logs and in Excel columns.

---

## 6. Handoff to frontend (when you come back)

When Phase B is usable:

1. Provide **OpenAPI** or a shared TypeScript package with request/response types.
2. Document **field mapping** from API JSON to current `Ticket` (or deliver the new canonical shape + migration notes).
3. Confirm **auth** for the SPA (e.g. delegated Graph vs client-credentials via your API only).

Then the frontend agent can execute the planned waves: tokens, shell, chart presets, KPI labeling, form deduplication, sheet split modules, and repository swap.

---

## 7. Related documents

- Frontend agent rule: [`.cursor/rules/frontend-agent.mdc`](../.cursor/rules/frontend-agent.mdc)
- Frontend modernization prompt: [`docs/AGENT_PROMPT_FRONTEND_MODERNIZATION.md`](./AGENT_PROMPT_FRONTEND_MODERNIZATION.md)
- **Per-version cross-team sync (branch-only, never `main`):** [`docs/coordination/README.md`](./coordination/README.md)

---

## 8. Per-release coordination with frontend

Use the branch-local pair `docs/coordination/vX.Y.Z-backend-sync.md` and `docs/coordination/vX.Y.Z-frontend-sync.md` on the same `v1.x.y` branch. **Do not merge** `docs/coordination/**` into `main` — see [`docs/coordination/README.md`](./coordination/README.md).
