# Cross-process coordination (`docs/coordination/`)

> **pt-BR:** Esta pasta guarda **Markdowns por versão** (`vX.Y.Z`) para o time de **front** e **back** se alinharem **na branch de trabalho**. **Nada aqui deve ir para `main`.** Antes de abrir ou fazer merge de PR para `main`, **remova** `docs/coordination/` do diff (ou abra um PR de código que não inclua estes arquivos). Os arquivos continuam no remoto na branch `v*` como registro vivo do processo.

---

## Policy (mandatory)

1. **Path:** only `docs/coordination/**` is covered by this policy (per-version sync files + this README).
2. **Purpose:** short-lived handoffs between frontend and backend workstreams on the **same** `v1.x.y` branch.
3. **Never merge into `main`.** The long-lived branch `main` must not contain version-sync Markdown. Reasons:
   - avoids stale process docs in production baseline;
   - avoids leaking work-in-progress commitments into the canonical tree;
   - keeps review focused on code shipped to users.
4. **Workflow:**
   - Copy `TEMPLATE-frontend-sync.md` → `vX.Y.Z-frontend-sync.md` and `TEMPLATE-backend-sync.md` → `vX.Y.Z-backend-sync.md` when cutting a new `v*` branch.
   - Update both files during the release line; link OpenAPI commits, schema decisions, and blockers.
   - Before merging the **code** PR to `main`, run:  
     `git rm -r docs/coordination`  
     (or exclude the folder when squashing / cherry-picking) so `main` stays clean.
5. **Remote:** pushing `docs/coordination/**` to `origin/v1.x.y` is **expected** so both agents can read the same handoff without relying on chat history.

---

## File naming convention

| File | Owner focus |
|------|-------------|
| `vX.Y.Z-frontend-sync.md` | UI, repository swap, dashboard, forms, sheet UX |
| `vX.Y.Z-backend-sync.md` | API, Excel/Graph, auth, schema, audit |

Templates: [`TEMPLATE-frontend-sync.md`](./TEMPLATE-frontend-sync.md), [`TEMPLATE-backend-sync.md`](./TEMPLATE-backend-sync.md).

---

## Related canonical docs (may merge to `main` when reviewed)

- [`../AGENT_PROMPT_FRONTEND_MODERNIZATION.md`](../AGENT_PROMPT_FRONTEND_MODERNIZATION.md)
- [`../BACKEND_EXCEL_HANDOFF.md`](../BACKEND_EXCEL_HANDOFF.md)
- [`.cursor/rules/frontend-agent.mdc`](../../.cursor/rules/frontend-agent.mdc)
