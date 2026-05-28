# Frontend engineering context — Access Requests Portal SPA

> **Single source of truth** for the React/TypeScript SPA that ships inside the
> `access-requests-portal` Databricks App. **English** technical standard. Read
> this document before touching any code under `apps/access-requests-portal/web/`
> in the ADO monorepo `data-platform-bees-consumer-mkt-strategy-insights`.
>
> **Companion docs:**
> - [`BACKEND_ENGINEERING_CONTEXT.md`](./BACKEND_ENGINEERING_CONTEXT.md) — server-side standards.
> - [`FRONTEND_API_CONTRACT.md`](./FRONTEND_API_CONTRACT.md) — HTTP contract, error envelope, field mapping.
> - [`DATABRICKS_APPS_TARGET_ARCHITECTURE.md`](./DATABRICKS_APPS_TARGET_ARCHITECTURE.md) — deploy topology.

---

## 1. TL;DR

- **One SPA, two homes.** Code lives in **ADO** at
  `apps/access-requests-portal/web/`. Canonical docs and `.cursor/rules/`
  live here in **GitHub**. Never invert the two.
- **Same-origin call** to the BFF (`/api/v1/...`). No CORS, no API keys in
  the browser. The SPA ships next to the FastAPI app in a single Databricks
  App bundle (`apps/access-requests-portal/static/`).
- **Dark-first**, indigo + cyan accents, dense layout, BEES-team visual
  identity (see §5). Light theme is a first-class toggle, persisted in
  `localStorage`, applied via `<html class="light">`.
- **English UI chrome**. Data values can still arrive in pt-BR from the BE
  (e.g. `status="Aberto"`); the SPA translates via `lib/status-i18n.ts`.
- **No PR auto-creation on ADO.** Push to a feature branch and stop — the
  human opens the PR. See §11.

---

## 2. Mandatory quality bar

| Practice | What it means here |
|----------|--------------------|
| **TypeScript strict** | `tsc -p tsconfig.json --noEmit` is the lint command (`npm run lint`). Zero errors before pushing. |
| **No `any` drift** | When backend types are missing, write a small adapter type next to the consumer — do not leak `any` across module boundaries. |
| **Hooks + repository pattern** | Components consume `useTickets`, `useTicketEvents`; data access goes through `HttpTicketsRepository`, never `fetch()` in components. |
| **Single source of truth per concern** | Status/priority labels live in `lib/status-i18n.ts`; phase ownership and SLA in `lib/phase-tracker.ts`; Jira key extraction in `lib/jira.ts`. Do not duplicate. |
| **A11y first** | `aria-sort`, `aria-haspopup`, `role="menu"`, focus-visible rings on all interactive elements. The drawer traps focus inside the modal. |
| **Print mode** | Every new UI element decides up-front whether it should appear in `@media print` (chrome hides, data prints). Add the `print:hidden` Tailwind utility on UI affordances by default. |

Non-goals: a state management library (the current footprint does not warrant Redux/Zustand/Jotai), runtime CSS-in-JS, server components.

---

## 3. Tech stack

| Layer | Choice | Notes |
|-------|--------|-------|
| Build | **Vite 6** | `node_modules/vite/bin/vite.js build`. Output goes to `web/dist/`, then is **manually** copied into `apps/access-requests-portal/static/` (the BFF serves that folder verbatim). |
| Framework | **React 19** | Functional components, hooks, `useId`, `useLayoutEffect`. |
| Language | **TypeScript 5** | `strict: true`. `lib: ["DOM","DOM.Iterable","ESNext"]`. |
| Styling | **Tailwind CSS v4** | Custom CSS overrides in `src/index.css` (light-theme tuning, print rules, animation keyframes). |
| Icons | **lucide-react** | Stick to lucide; do not mix icon libraries. |
| Type generation | **openapi-typescript** | `npm run generate:api` hits `http://127.0.0.1:8000/api/v1/openapi.json` and writes `src/lib/api-types.generated.ts`. `src/lib/api-types.ts` re-exports from there plus the SPA-only `FilterQuery` and `TicketsListQuery`. |

There is **no `npm` on the agent machine**. Run scripts via Cursor's bundled
Node binary:

```powershell
$node = "C:\Program Files\cursor\resources\app\resources\helpers\node.exe"
& $node node_modules\typescript\bin\tsc -p tsconfig.json --noEmit   # lint
& $node node_modules\vite\bin\vite.js build                          # build
& $node node_modules\vite\bin\vite.js                                # dev
```

---

## 4. Folder layout (the only one that matters)

```
apps/access-requests-portal/
├── web/                              # SPA source
│   ├── src/
│   │   ├── App.tsx                   # orchestrator: state, repo, modals, drawer
│   │   ├── main.tsx                  # ReactDOM.createRoot
│   │   ├── index.css                 # Tailwind + custom overrides (light, print, anim)
│   │   ├── contexts/
│   │   │   ├── ThemeContext.tsx      # dark/light, localStorage, <html class="light">
│   │   │   └── ToastContext.tsx      # toast queue + ToastContainer
│   │   ├── lib/
│   │   │   ├── api-types.ts          # re-exports from generated + SPA-only types
│   │   │   ├── api-types.generated.ts# DO NOT EDIT — openapi-typescript output
│   │   │   ├── repositories/
│   │   │   │   ├── TicketsRepository.ts     # port
│   │   │   │   └── HttpTicketsRepository.ts # adapter (fetch)
│   │   │   ├── useTickets.ts         # query/loadMore/create/patch/remove/bulk
│   │   │   ├── useTicketEvents.ts    # audit pagination
│   │   │   ├── status-i18n.ts        # status/priority pt-BR ↔ English
│   │   │   ├── action-kind.ts        # grant/revoke prefix (workaround; see §8)
│   │   │   ├── phase-tracker.ts      # owner + SLA + traffic light
│   │   │   ├── jira.ts               # extractJiraKey
│   │   │   ├── ticket-age.ts         # formatRelative + isTerminalStatus
│   │   │   ├── sort-tickets.ts       # client-side, stable, tri-state cycle
│   │   │   └── tickets-query.ts      # URL <-> FilterQuery
│   │   └── components/
│   │       ├── layout/{Sidebar,Header}.tsx
│   │       ├── ui/                   # primitives (StatusBadge, Modal, Skeleton,
│   │       │                         #             ActionMenu, AgeBadge,
│   │       │                         #             JiraBadge, ActionBadge,
│   │       │                         #             PhaseBadge, ThemeToggle)
│   │       ├── TicketsTable.tsx      # row rendering + sort headers + selection
│   │       ├── TicketDrawer.tsx      # Details / Edit / Audit tabs
│   │       ├── CreateTicketForm.tsx  # used for both create and clone-to-edit
│   │       ├── ConfirmDeleteModal.tsx# centralised soft-delete UX
│   │       ├── FiltersBar.tsx        # collapsible, chips when collapsed
│   │       ├── BulkToolbar.tsx       # multi-select bulk-update
│   │       ├── Pagination.tsx        # "Load more" + count
│   │       ├── EventsTimeline.tsx    # Audit tab
│   │       ├── PhaseTimeline.tsx     # Details tab — vertical SLA breakdown
│   │       ├── PrintReportHeader.tsx # cover page for the print output
│   │       └── {EmptyState,ErrorBanner,HealthIndicator}.tsx
│   ├── package.json
│   ├── tsconfig.json + tsconfig.node.json
│   ├── vite.config.ts                # dev proxy: /api → http://127.0.0.1:8000
│   ├── index.html                    # inline FOUC-prevention script
│   └── README.md
└── static/                           # serve-as-is bundle
    ├── index.html                    # COPY of web/dist/index.html
    ├── assets/                       # COPY of web/dist/assets/*  (hashed)
    └── _smoke.html                   # backend agent's smoke console — DO NOT TOUCH
```

> **Hard rule:** the FE branch may only edit files under `web/` and the
> hashed bundle under `static/assets/` + `static/index.html`. Never touch
> `_smoke.html` — that belongs to the backend branch's smoke harness.

---

## 5. Visual identity & design system

- **Dark-first.** Default `<html>` has no class; `.light` toggles light
  theme. Theme state lives in `ThemeContext` and persists to
  `localStorage.access-portal-theme`. An inline script in `index.html`
  applies the class before React mounts to avoid FOUC.
- **Palette.** Slate-900 surfaces, indigo-500/cyan-500 accents for primary
  actions, emerald/amber/rose for SLA traffic lights, rose for destructive.
- **Density.** `text-sm` for body, `text-[11px]` for metadata, `font-mono`
  for ticket IDs, row_version and Jira keys. Rows pack tight (`py-2.5`).
- **Action accents.** Grant rows render as-is. Revoke rows get a rose left
  border via `data-action-kind="revoke"` on `<tr>` (see `index.css`).
- **Phase ownership.** The "Time open" column is **not** a single timer —
  it is a per-phase traffic light that names the accountable team:
  - To Do → **Martech** (us). SLA: ≤ 2h green, ≤ 8h yellow, > 8h red.
  - In Progress / Pending → **BEES** team. SLA: ≤ 24h / ≤ 72h / > 72h.
  - Resolved / Cancelled → terminal, no SLA, no traffic light.
  - **The owner label is `BEES`, not `BIS`.** A stale `bis` literal anywhere
    in `web/src` is a regression — `rg -n '\bbis\b|BIS' apps/access-requests-portal/web/src`
    must return zero matches.
- **Spreadsheet print.** `@media print` hides sidebar, header chrome,
  drawer, modals, sort icons, the three-dot menu, and the SIT/UAT badge.
  The table grows to 100%, gets full cell borders and plain values. A
  `PrintReportHeader` cover page lists the export time, filters and row
  count.

---

## 6. State & data flow

```
HttpTicketsRepository  ←─ same-origin fetch (X-Request-ID, JSON, 409 mapping)
        ▲
        │
   useTickets (query, loadMore, optimistic pending, create/patch/remove, bulk)
   useTicketEvents (cursor pagination)
        ▲
        │
   App.tsx (selected ticket, sort, filters, pending delete, toast)
        ▲
        │
   Components (presentational; receive callbacks)
```

Standing patterns:

- **Query → URL.** `tickets-query.ts` (`filterQueryFromSearch` +
  `filterQueryToSearch`) round-trips the filter state through
  `history.replaceState`. The user can paste a URL and land on the same
  filtered view.
- **Optimistic pending tickets.** `useTickets.create` inserts a row with
  `_clientRequestId` immediately; on failure it stays in a "Retry / Discard"
  banner instead of disappearing. `TicketsTable` renders pending rows with
  reduced opacity.
- **409 UX.** Any write returning `409` triggers `repo.get(ticket_id)` to
  refresh the row, surfaces a toast "Row updated by another user", and
  keeps the drawer/form open with a banner instead of losing the user's
  input.
- **Soft-delete is centralised.** `ConfirmDeleteModal` is mounted at the
  App level. Both the row-level three-dot menu and the drawer footer just
  set `pendingDelete` → the modal handles confirm, errors and 409.
- **Sort is client-side.** `sort-tickets.ts` operates on the currently
  loaded page (tri-state cycle: none → asc → desc → none). Server-side
  ordering will be needed once we paginate beyond a few hundred rows.

---

## 7. Component conventions

- **Portal-based popovers.** `components/ui/ActionMenu.tsx` renders through
  `createPortal(menuNode, document.body)`. **Do not revert this** — the
  tickets table combines `overflow: hidden` and `overflow: auto`, and the
  page header uses `backdrop-filter`, both of which trap `position: fixed`.
  Placement is viewport-aware (auto-flip vertical, clamp horizontal).
  Default `align='bottom-right'`; the drawer footer opts in to
  `align='top-right'`.
- **Modals.** `components/ui/Modal.tsx` is the only modal primitive.
  Backdrop click + Escape close; focus is trapped inside; portal-mounted.
- **Status & priority badges.** `StatusBadge` always goes through
  `statusLabel()` / `priorityLabel()`. Never hard-code "Aberto" / "Em
  Atendimento" in JSX.
- **Forms.** `CreateTicketForm` runs in two modes: `create` (default) and
  `edit`. In `create` mode it also accepts an `initialValue` ticket for
  the **clone** workflow (resets `status`, `opened_on`, `submitted_at`,
  `reference_url`; copies everything else).
- **Toasts.** `useToast()` from `ToastContext`. Tones: `success`, `error`,
  `warning`, `info`. Auto-dismiss 5s; pinned errors live until clicked.
- **Skeletons.** `Skeleton` and `SkeletonPulse` render during the first
  fetch — never spinners.

---

## 8. Tactical workarounds (track these for follow-up)

| What | Where | Why | Removal trigger |
|------|-------|-----|-----------------|
| `extra = "ACTION:REVOKE[;<note>]"` encodes the grant/revoke action kind. | `lib/action-kind.ts`, `CreateTicketForm`, `TicketsTable` (left-border accent). | The original schema did not have an `action_kind` column. | The backend agent's branch `agent/access-requests-portal-backend` (commit `49d2617`) adds the typed `action_kind` column. Once deployed, swap FE to read/write the typed field and keep the prefix as a read-only fallback for one release. |
| `employee_id = "EXT:<company>"` encodes a third-party requester + their vendor name. | `CreateTicketForm` (radio buttons), reads in `RequesterCell`. | Same reason — no typed `requester_kind` / `requester_company` columns yet. | Same backend branch ships `requester_kind` and `requester_company`. Same plan: swap FE, keep prefix fallback for a release, then drop. |
| Client-side sort. | `lib/sort-tickets.ts`. | The BFF does not yet expose `order_by`. | Add `order_by` to `GET /tickets` on the BE, then push the sort to the server when the list exceeds one page. |
| FE-derived "Time open" and "phase". | `lib/phase-tracker.ts`, `PhaseBadge`, `PhaseTimeline`. | Avoided new BE columns; reconstructed from `updated_at` (table approximation) and the audit event stream (drawer truth). | If product wants exact per-phase SLAs in BI/exports, surface a `phase_breakdown` field from BE. |

---

## 9. Build, deploy & verification

### 9.1 Local dev

```powershell
$node = "C:\Program Files\cursor\resources\app\resources\helpers\node.exe"
cd apps\access-requests-portal\web
& $node node_modules\vite\bin\vite.js          # dev server on :5173, proxies /api to :8000
```

Run the BFF in a second terminal so the proxy resolves
(`python -m uvicorn app:app --reload`, see backend docs).

### 9.2 Production build & sync

```powershell
& $node node_modules\typescript\bin\tsc -p tsconfig.json --noEmit
& $node node_modules\vite\bin\vite.js build

# Sync to the bundle the BFF serves. Never touch _smoke.html.
Remove-Item ..\static\assets\* -Force
Copy-Item dist\index.html ..\static\index.html -Force
Copy-Item dist\assets\* ..\static\assets\ -Force
```

The deploy is a normal ADO PR merge; CI on master rebuilds the Databricks
App bundle.

### 9.3 Verification checklist before every push

- [ ] `tsc --noEmit` clean.
- [ ] `vite build` clean — note the new hash names in the commit message.
- [ ] `_smoke.html` untouched (`git diff -- apps/access-requests-portal/static/_smoke.html` empty).
- [ ] No `bis` / `BIS` in `web/src` (`rg -n '\bbis\b|BIS' apps/access-requests-portal/web/src`).
- [ ] No `console.log` left behind.
- [ ] No pt-BR strings in UI chrome (use `lib/status-i18n.ts` for data values).

---

## 10. Cross-repo coordination

| Topic | GitHub `gestao_lancamento_de_chamados` | ADO `data-platform-bees-consumer-mkt-strategy-insights` |
|-------|----------------------------------------|---------------------------------------------------------|
| Owns | Canonical docs (`docs/`), `.cursor/rules/`, legacy SPA history. | Production code (`apps/access-requests-portal/`), CI, Databricks deploy. |
| Default agent branch (docs) | `agent/backend-docs` (reset to `origin/main` after merge). A frontend equivalent — `agent/frontend-docs` — can be cut from `main` when a doc-only FE wave starts. | n/a |
| Default agent branch (code) | n/a | `agent/access-requests-portal-backend` for BE; FE uses ad-hoc `feat/access-requests-portal-frontend-*` or `fix/...` cut from `master`. |
| PR policy | Backend doc PRs go to `main`. No auto-PR. | **Agent MAY commit and push, MUST NOT open PRs automatically.** Humans open PRs. |
| Conventional commits | Required, English. | Required, English. |
| Coordination notes | `docs/coordination/v<x.y.z>-{backend,frontend}-sync.md` for cross-repo waves. | Mirror the same wave id in the PR title. |

---

## 11. Branching & commits (FE-specific)

- **Cut from `origin/master`**, not from a sibling feature branch (the
  v1.x.y branches live in GitHub only — they do not exist on ADO).
- Naming: `feat/access-requests-portal-frontend-<topic>` or
  `fix/access-requests-portal-frontend-<topic>`. Use `fix/` for any change
  that doesn't add a user-visible capability.
- One commit per logical unit. Build outputs (`static/`) stay in the **same
  commit** as the `web/src` changes that produced them — never split.
- Conventional Commits, English. Example template:

  ```
  fix(access-requests-portal): row menu anchors to the three-dot trigger

  * web/src/components/ui/ActionMenu.tsx: <what / why>
  * static/index.html + static/assets/*: rebuilt bundle.
  ```

- Push with `git push origin <branch>`. Do **not** run `gh pr create` or
  `az repos pr create`. Provide the user with the prefilled ADO URL
  (`https://dev.azure.com/ab-inbev/GHQ_B2B_Delta/_git/data-platform-bees-consumer-mkt-strategy-insights/pullrequestcreate?sourceRef=<branch>&targetRef=master&title=<urlencoded>`)
  plus a description block they can paste.

---

## 12. Known gotchas

- **`position: fixed` can be trapped.** Anything inside the table shell or
  the page header (which has `backdrop-filter`) needs a portal-rendered
  popover. The `ActionMenu` already does this; copy that pattern when you
  add any new popover/tooltip/dropdown that anchors outside the trigger.
- **Vite hashes shift CSS even when only TS changed.** Tailwind's
  utility-purging picks up new classes from `.tsx` edits. Always re-sync
  `static/assets/` after the build; never hand-edit a hashed file.
- **OneDrive paths break `npm` heuristics.** Always pass absolute paths to
  Node and prefer `-LiteralPath` in PowerShell. Quote paths with spaces.
- **Commit message files must not contain a BOM.** Use
  `[System.IO.File]::WriteAllText($f, $msg, [System.Text.UTF8Encoding]::new($false))`
  before `git commit -F $f`.
- **Sticky `<thead>` has `backdrop-blur`.** It creates a stacking context.
  Anything that should appear above rows but below the page header must
  pick z-index carefully (`z-10` thead, `z-40` sidebar, `z-50` modals/portals).
- **Stash leftover from a previous session can pollute the build.** If
  `git status` shows files you did not change (e.g. `EventsTimeline.tsx`
  modified, `event-diff.ts` untracked), stash them with an explicit
  pathspec before building so Tailwind doesn't pick up extra classes.

---

## 13. Hand-off checklist (full-stack agent)

When the new agent picks up this codebase:

1. Clone both repos. Treat the GitHub one as documentation, the ADO one as
   the deploy target. Never edit production code in GitHub.
2. Read **this file** and `FRONTEND_API_CONTRACT.md` end-to-end before
   writing any code.
3. `cd apps/access-requests-portal/web && <node> node_modules/vite/bin/vite.js`
   — confirm the dev server boots and the proxy reaches the BFF at `:8000`.
4. Open the SPA in the browser; toggle the theme; create + clone + delete a
   ticket; trigger a 409 by editing the same ticket in two tabs; verify
   the per-phase traffic light reflects the current status.
5. Run `<node> node_modules/typescript/bin/tsc --noEmit` and
   `<node> node_modules/vite/bin/vite.js build` once before touching
   anything — establish a clean baseline.
6. Pick up open follow-ups from §8 (action_kind / requester_kind FE
   migration once the BE columns are live in SIT).

---

## 14. Summary

**Default stack for the SPA:** **Vite + React 19 + TypeScript strict +
Tailwind v4**, with a **repository port** for HTTP, **hooks** for state,
**portal-based popovers** for any anchored UI, and a **single-translation
layer** for backend Portuguese values. Enforce the quality bar in §2,
honour the branching rules in §11, and resist adding state libraries or
new abstractions until the next concrete use case forces them.

The HTTP contract this UI talks to lives in
[`FRONTEND_API_CONTRACT.md`](./FRONTEND_API_CONTRACT.md). The Python BFF
that serves it lives in
[`BACKEND_ENGINEERING_CONTEXT.md`](./BACKEND_ENGINEERING_CONTEXT.md). Keep
all three documents in sync whenever a wave changes the contract surface.
