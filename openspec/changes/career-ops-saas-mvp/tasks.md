# Tasks: career-ops-saas MVP (Fase 1)

_Change:_ career-ops-saas-mvp
_Status:_ ready
_Date:_ 2026-06-03
_Spec:_ openspec/changes/career-ops-saas-mvp/spec.md
_Design:_ openspec/changes/career-ops-saas-mvp/design.md

---

## Review Workload Forecast

| Metric | Estimate |
|--------|----------|
| New files | ~85 |
| Estimated changed lines | ~2,600 |
| 400-line budget risk | **HIGH** |
| Chained PRs recommended | **Yes** |
| Decision needed before apply | **Yes** |

**Recommended PR splits (stacked-to-main):**
- PR-1: Milestone 1 — Repo + Infrastructure
- PR-2: Milestone 2 + 4 — Go API auth, middleware, WebSocket
- PR-3: Milestone 3 — Go API domain handlers
- PR-4: Milestone 5 + 6 — Node.js worker
- PR-5: Milestone 7 — Next.js frontend
- PR-6: Milestone 8 — Integration + RLS tests

---

## Milestone 1: Repo + Infrastructure

- [x] T-01: Init monorepo root con `api/`, `worker/`, `web/`, `db/`, `README.md` — `career-ops-saas/`
- [x] T-02: Write `.env.example` con todas las vars (DATABASE_URL, JWT_SECRET, GOOGLE_*, R2_*, ANTHROPIC_API_KEY, SERPER_API_KEY) — `.env.example`
- [x] T-03: Write `docker-compose.yml` — postgres, api, worker, web con healthchecks — `docker-compose.yml`
- [x] T-04: Write `db/schema.sql` — enums, 8 tablas, indexes, UNIQUE constraints — `db/schema.sql`
- [x] T-05: Write `db/rls.sql` — 8 RLS policies + `FORCE ROW LEVEL SECURITY` on `app_user` — `db/rls.sql`
- [x] T-06: Write `db/auth_upsert_user.sql` — función `SECURITY DEFINER auth_upsert_user(email, google_id)` — `db/auth_upsert_user.sql`
- [x] T-07: Write `db/migrations/001_initial.sql` — compone schema + RLS + función en orden — `db/migrations/001_initial.sql`
- [x] T-08: Write `db/queries/*.sql` — queries sqlc para todas las tablas — `db/queries/`
- [x] T-09: Write `db/sqlc.yaml` — apunta a queries, schema, output package `internal/db` — `db/sqlc.yaml`
- [~] T-10: Run `sqlc generate` — PENDING: sqlc not installed. Run `go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest` then `cd db && sqlc generate` — `api/internal/db/` (output dir created)

---

## Milestone 2: Go API — Auth + Core

- [x] T-11: Init `go.mod` — chi, pgx/v5, golang-jwt, golang.org/x/oauth2, godotenv — `api/go.mod`
- [x] T-12: Write `internal/config/config.go` — carga y valida todas las env vars — `api/internal/config/config.go`
- [x] T-13: Write `internal/platform/postgres.go` — pgxpool init, helper `TenantQuery` que setea `app.current_user_id` — `api/internal/platform/postgres.go`
- [x] T-14: Write `internal/platform/r2.go` — S3-compatible client, `SignedDownloadURL(key, 24h)` — `api/internal/platform/r2.go`
- [x] T-15: Write `internal/queue/boss.go` — INSERT en `pgboss.job`, `Enqueue(jobType, payload)` — `api/internal/queue/boss.go`
- [x] T-16: Write `internal/middleware/auth.go` — validación Bearer JWT, inyecta `user_id` en ctx, 401 en expirado — `api/internal/middleware/auth.go`
- [x] T-17: Write `internal/middleware/tenant.go` — lee `user_id` del ctx, ejecuta `SET LOCAL app.current_user_id` — `api/internal/middleware/tenant.go`
- [x] T-18: Write `internal/middleware/logging.go`, `recover.go`, `cors.go` — `api/internal/middleware/`
- [x] T-19: Write `internal/auth/jwt.go` — `IssueAccessToken`, `IssueRefreshToken`, `VerifyAccessToken` HS256 — `api/internal/auth/jwt.go`
- [x] T-20: Write `internal/auth/oauth.go` — Google OAuth2 code exchange + userinfo — `api/internal/auth/oauth.go`
- [x] T-21: Write `internal/auth/service.go` + `handler.go` — GET /auth/google, callback con upsert SECURITY DEFINER, POST /auth/refresh con rotación — `api/internal/auth/`
- [x] T-22: Write `cmd/api/main.go` — wiring chi router, middleware stack, listen — `api/cmd/api/main.go`

---

## Milestone 3: Go API — Domain Handlers

*(T-23..T-25, T-27..T-28, T-30..T-31, T-33..T-34, T-36..T-37, T-38 son paralelizables por dominio)*

- [ ] T-23: Write `internal/jobs/repo.go` — ListByUser, GetByID, Insert, UpdateStatus — `api/internal/jobs/repo.go`
- [ ] T-24: Write `internal/jobs/service.go` — manual URL add, detección de plataforma — `api/internal/jobs/service.go`
- [ ] T-25: Write `internal/jobs/handler.go` — GET /api/jobs, POST /api/jobs, GET /api/jobs/:id — `api/internal/jobs/handler.go`
- [ ] T-26: Wire jobs routes en `main.go` con auth + tenant middleware — `api/cmd/api/main.go`
- [ ] T-27: Write `internal/companies/service.go` — resolución de provider_id desde careers_url — `api/internal/companies/service.go`
- [ ] T-28: Write `internal/companies/handler.go` — GET/POST/DELETE /api/companies — `api/internal/companies/handler.go`
- [ ] T-29: Wire companies routes en `main.go` — `api/cmd/api/main.go`
- [ ] T-30: Write `internal/scan/service.go` — INSERT scan_run, fan-out pg-boss "scan-company" jobs — `api/internal/scan/service.go`
- [ ] T-31: Write `internal/scan/handler.go` — POST /api/scan → 202 con scan_run_id — `api/internal/scan/handler.go`
- [ ] T-32: Wire scan route en `main.go` — `api/cmd/api/main.go`
- [ ] T-33: Write `internal/evaluate/service.go` — usage check (free plan cap), enqueue "evaluate-job" — `api/internal/evaluate/service.go`
- [ ] T-34: Write `internal/evaluate/handler.go` — POST /api/jobs/:id/evaluate, GET /api/jobs/:id/report — `api/internal/evaluate/handler.go`
- [ ] T-35: Wire evaluate + report routes en `main.go` — `api/cmd/api/main.go`
- [ ] T-36: Write `internal/cv/service.go` — enqueue "generate-pdf", signed R2 URL — `api/internal/cv/service.go`
- [ ] T-37: Write `internal/cv/handler.go` — POST /api/jobs/:id/cv, GET /api/jobs/:id/cv — `api/internal/cv/handler.go`
- [ ] T-38: Write `internal/tracker/service.go` + `handler.go` — GET /api/applications, PATCH /api/applications/:id — `api/internal/tracker/`

---

## Milestone 4: Go API — WebSocket

- [x] T-39: Write `internal/ws/hub.go` — registry `(user_id, scan_run_id)`, `Register`, `Unregister`, `Broadcast` — `api/internal/ws/hub.go`
- [x] T-40: Write `internal/ws/handler.go` — GET /ws/scan?token= WebSocket upgrade con JWT auth — `api/internal/ws/handler.go`
- [x] T-41: Write `internal/ws/listener.go` — goroutine `LISTEN scan_progress`, parse NOTIFY, llama `hub.Broadcast` — `api/internal/ws/listener.go`
- [x] T-42: Arrancar goroutine listener en `main.go`, wire `/ws/scan` route — `api/cmd/api/main.go`
- [x] T-43: Verificar envelope WS (`event`, `scan_run_id`, `ts`, `data`) contra design spec — `api/internal/ws/hub.go`
- [x] T-44: Write `api/Dockerfile` — multi-stage Go build, distroless final — `api/Dockerfile`

---

## Milestone 5: Node.js Worker — Core

*(T-46..T-52 paralelizables; T-53 requiere T-46 + T-47)*

- [ ] T-45: Init `package.json` — pg-boss, pg, @anthropic-ai/sdk, @aws-sdk/client-s3, express, dotenv — `worker/package.json`
- [ ] T-46: Write `lib/db.mjs` — pg Pool, `tenantQuery(userId, sql, params)` con `SET LOCAL` — `worker/lib/db.mjs`
- [ ] T-47: Write `lib/queue.mjs` — pg-boss instance, `registerWorker(jobType, handler, opts)` — `worker/lib/queue.mjs`
- [ ] T-48: Write `lib/anthropic.mjs` — SDK client, `evaluate(systemBlocks, userContent)` con cache headers, 120s timeout — `worker/lib/anthropic.mjs`
- [ ] T-49: Write `lib/r2.mjs` — S3-compatible R2 upload, `uploadBuffer(key, buf, mimeType)` — `worker/lib/r2.mjs`
- [ ] T-50: Write `lib/prompt.mjs` — builds prompt desde `modes/oferta.template.md` + CV + profile_json del DB — `worker/lib/prompt.mjs`
- [ ] T-51: Write `lib/progress.mjs` — `notify(client, scanRunId, event, data)` → `pg_notify` — `worker/lib/progress.mjs`
- [ ] T-52: Copiar providers verbatim desde career-ops: `_http.mjs`, `_types.js`, `greenhouse.mjs`, `ashby.mjs`, `lever.mjs`, `recruitee.mjs`, `smartrecruiters.mjs`, `workable.mjs` — `worker/providers/`
- [ ] T-53: Write `index.mjs` — boot pg-boss, registrar workers, Express health en :3001 — `worker/index.mjs`

---

## Milestone 6: Node.js Worker — Jobs

*(T-54..T-56 secuencial; T-57..T-58 secuencial; T-59..T-60 secuencial — las 3 cadenas son paralelas entre sí)*

- [ ] T-54: Write `jobs/scan.mjs` — consume "scan-company": cargar provider, fetch, UPSERT jobs, NOTIFY scan.job_found + scan.company.done — `worker/jobs/scan.mjs`
- [ ] T-55: Add scan run aggregation en `jobs/scan.mjs` — UPDATE scan_runs.status a completed/partial en último company — `worker/jobs/scan.mjs`
- [ ] T-56: Handle Workable graceful degrade — catch error, NOTIFY scan.company.error, append errors_json, no fallar siblings — `worker/jobs/scan.mjs`
- [ ] T-57: Write `jobs/evaluate.mjs` — consume "evaluate-job": fetch job + CV + profile, Anthropic API, parsear bloques A-G, INSERT applications + reports, UPDATE usage — `worker/jobs/evaluate.mjs`
- [ ] T-58: Add parse error guard en `jobs/evaluate.mjs` — on failure persiste `{"parse_error": true}`, nunca pierde la fila — `worker/jobs/evaluate.mjs`
- [ ] T-59: Write `shared/generate-pdf.mjs` — refactor como async `renderPDF(htmlContent)` export con Playwright `--no-sandbox` — `worker/shared/generate-pdf.mjs`
- [ ] T-60: Write `jobs/pdf.mjs` — consume "generate-pdf": fetch report + CV, render HTML, Playwright PDF, upload R2, UPDATE applications.pdf_path — `worker/jobs/pdf.mjs`
- [ ] T-61: Copiar `shared/liveness-core.mjs` verbatim desde career-ops — `worker/shared/liveness-core.mjs`
- [ ] T-62: Write `worker/Dockerfile` — node:20 + Playwright Chromium, `--no-sandbox`, shm 1gb — `worker/Dockerfile`

---

## Milestone 7: Next.js Frontend

*(T-64+T-65 paralelo; T-66..T-72 paralelos entre sí después de T-64+T-65)*

- [ ] T-63: Init Next.js 14 app con TypeScript, Tailwind, shadcn/ui — `web/`
- [ ] T-64: Write `lib/api.ts` — fetch wrapper con Authorization header, interceptor refresh 401 — `web/lib/api.ts`
- [ ] T-65: Write `lib/auth.ts` — token storage (httpOnly cookie), `useAuth` hook, auth guard — `web/lib/auth.ts`
- [ ] T-66: Write login page `/login` + callback `/auth/callback` — `web/app/login/page.tsx`, `web/app/auth/callback/page.tsx`
- [ ] T-67: Write dashboard `app/page.tsx` — job list paginada (received_at desc), form Add Job URL, badges de status — `web/app/page.tsx`
- [ ] T-68: Write job detail panel `app/jobs/[id]/page.tsx` — info del job, botones Evaluate + Generate CV, display de bloques del reporte — `web/app/jobs/[id]/page.tsx`
- [ ] T-69: Write hook `hooks/useScanProgress.ts` — conecta a `/ws/scan?token=`, parsea eventos, actualiza job list en real-time — `web/hooks/useScanProgress.ts`
- [ ] T-70: Wire scan en dashboard — botón Scan Now → POST /api/scan → abre WS → toasts por scan.company.done/error — `web/app/page.tsx`
- [ ] T-71: Write tracker `app/tracker/page.tsx` — tabla de applications con status select, edición de notes, link descarga PDF — `web/app/tracker/page.tsx`
- [ ] T-72: Write companies page `app/companies/page.tsx` — lista watched companies, Add Company form (nombre, careers_url, provider_id), Remove — `web/app/companies/page.tsx`

---

## Milestone 8: Integration + Tests (MANDATORIO antes de ship)

- [ ] T-73: Write `db/tests/rls_test.sql` — pgTAP: usuario A no puede leer jobs/applications/reports/cvs de usuario B (ADR-3) — `db/tests/rls_test.sql`
- [ ] T-74: Write Go integration tests auth — `TestGoogleCallbackCreatesUser`, `TestRefreshTokenRotation`, `TestExpiredAccessToken401` — `api/internal/auth/handler_test.go`
- [ ] T-75: Write Go integration test cross-tenant — `TestUserBCannotReadUserAJob` → assert 403/404 — `api/internal/jobs/handler_test.go`
- [ ] T-76: Write Node.js integration test graceful degrade — mock Workable 404, assert scan_runs.status = partial, errores en errors_json, jobs de Greenhouse insertados — `worker/tests/scan.test.mjs`
- [ ] T-77: Write `scripts/e2e-smoke.sh` — docker compose up, auth flow, POST /api/jobs, POST /api/scan, esperar WS event, assert 200 /api/applications — `scripts/e2e-smoke.sh`
- [ ] T-78: Verificar Docker Compose healthcheck de postgres dispara antes que api/worker arranquen — `docker-compose.yml`

---

## Dependency Graph

```
M1 (T-01..T-10)
  ├── M2 (T-11..T-22)  ← requiere T-10
  │     ├── M3 (T-23..T-38)  ← requiere M2
  │     └── M4 (T-39..T-44)  ← requiere M2
  └── M5 (T-45..T-53)  ← requiere T-01
        └── M6 (T-54..T-62)  ← requiere M5

M2 stable
  └── M7 (T-63..T-72)  ← paralelo a M3/M4/M5/M6

M3 + M4 + M6 + M7 listos
  └── M8 (T-73..T-78)
```

**Total: 78 tareas — 6 PRs stacked-to-main**
