# Apply Progress: career-ops-saas-mvp

## PR-1: Milestone 1 — Repo + Infrastructure
_Completed: 2026-06-04_

### Completed
- [x] T-01: Monorepo structure (`api/`, `worker/`, `web/`, `db/migrations/`, `db/queries/`, updated `README.md`)
- [x] T-02: `.env.example` with all required variables (DATABASE_URL, JWT_SECRET, JWT_REFRESH_SECRET, GOOGLE_*, R2_*, ANTHROPIC_API_KEY, ANTHROPIC_MODEL, SERPER_API_KEY, WEB_ORIGIN, PORT, WORKER_PORT)
- [x] T-03: `docker-compose.yml` — postgres:16 with healthcheck, api (depends_on healthy postgres), worker (depends_on healthy postgres, shm_size 1gb), web (depends_on api)
- [x] T-04: `db/schema.sql` — pgcrypto extension, 4 enums (plan_t, job_status_t, app_status_t, scan_status_t), 8 tables with exact column definitions and all indexes from design
- [x] T-05: `db/rls.sql` — 8 ENABLE ROW LEVEL SECURITY + 8 FORCE ROW LEVEL SECURITY + 8 CREATE POLICY statements using current_setting('app.current_user_id', true)::uuid
- [x] T-06: `db/auth_upsert_user.sql` — SECURITY DEFINER function for OAuth upsert that bypasses RLS
- [x] T-07: `db/migrations/001_initial.sql` — composes schema + RLS + auth function inline; also creates app_user role and grants DML
- [x] T-08: `db/queries/*.sql` — sqlc query files for all 8 tables:
  - `users.sql`: GetUserByGoogleID, GetUserByID, UpdateUserPlan, UpdateUserCVMarkdown, UpdateUserProfileJSON
  - `jobs.sql`: ListJobsByUser (paginated), GetJobByID, InsertJob, UpdateJobStatus, UpdateJobEvaluationJSON, UpsertJobByURL
  - `watched_companies.sql`: ListWatchedCompaniesByUser, GetWatchedCompanyByID, InsertWatchedCompany, DeleteWatchedCompany, ListEnabledWatchedCompaniesByUser
  - `applications.sql`: GetApplicationByJobID, InsertApplication, UpdateApplicationStatus, UpdateApplicationNotes, UpdateApplicationPDFPath, ListApplicationsByUser (paginated)
  - `reports.sql`: GetReportByApplicationID, InsertReport
  - `cvs.sql`: ListCVsByUser, GetMasterCVByUser, InsertCV, UpdateCV, SetMasterCV
  - `scan_runs.sql`: InsertScanRun, UpdateScanRunStatus, UpdateScanRunNewJobs, AppendScanRunError, GetScanRunByID
  - `usage.sql`: UpsertIncrementEvaluations, UpsertIncrementPDFs, GetUsageByUserMonth
- [x] T-09: `db/sqlc.yaml` — engine postgresql, queries: queries/, schema: schema.sql, out: ../api/internal/db, emit_json_tags: true
- [~] T-10: **PENDING — requires sqlc install**
  - sqlc is not installed in the environment
  - Output directory `api/internal/db/` created with instructions placeholder
  - To complete: `go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest` then `cd db && sqlc generate`

---

## PR-2: Milestone 2 + Milestone 4 — Go API Auth, Config, Middleware, JWT, OAuth2, WebSocket
_Completed: 2026-06-04_

### Completed
- [x] T-11: `api/go.mod` — go module init `github.com/miguel-anay/career-ops-saas/api` with deps: chi/v5, pgx/v5, golang-jwt/jwt/v5, golang.org/x/oauth2, godotenv, aws-sdk-go-v2 (+ config, s3), gorilla/websocket, google/uuid, sqlc-dev/pqtype
- [x] T-12: `api/internal/config/config.go` — `Config` struct + `Load()`, validates required env vars (DATABASE_URL, JWT_SECRET, JWT_REFRESH_SECRET, GOOGLE_*, PORT defaults to :8080)
- [x] T-13: `api/internal/platform/postgres.go` — `NewPool(ctx, databaseURL)`, `WithTenant(ctx, pool, userID, fn)` acquires conn + sets SET LOCAL app.current_user_id
- [x] T-14: `api/internal/platform/r2.go` — `NewR2Client(cfg)`, `SignedDownloadURL(key, expiry)`, `UploadObject(ctx, key, body, contentType)` — custom R2 endpoint resolver
- [x] T-15: `api/internal/queue/boss.go` — `Enqueue(ctx, pool, job Job)` inserts into pgboss.job with state='created'
- [x] T-16: `api/internal/middleware/auth.go` — `Authenticator(jwtSecret)` middleware, Bearer JWT validation, `GetUserID(ctx)` helper
- [x] T-17: `api/internal/middleware/tenant.go` — `TenantIsolation(pool)` middleware, reads user_id from ctx (set by Authenticator), validates pool
- [x] T-18: `api/internal/middleware/logging.go`, `recover.go`, `cors.go` — slog structured logging, panic recovery → 500 JSON, CORS with credentials
- [x] T-19: `api/internal/auth/jwt.go` — `IssueAccessToken` (1h), `IssueRefreshToken` (7d), `VerifyToken` — HS256
- [x] T-20: `api/internal/auth/oauth.go` — `NewOAuthConfig`, `ExchangeCode`, `GetUserInfo` → `GoogleUser{ID, Email, Name}`
- [x] T-21: `api/internal/auth/service.go` + `handler.go` — `UpsertUser` via SECURITY DEFINER function, `IssueTokenPair`, `GetUserByID`; handlers: GET /auth/google (CSRF state cookie), GET /auth/google/callback (exchange + upsert + redirect with access_token), POST /auth/refresh (rotation), POST /auth/logout
- [x] T-22: `api/cmd/api/main.go` — wires config, pgxpool, R2 client, chi router, global middleware (recover+logger+CORS), /auth routes, /api routes (auth+tenant), /ws routes, graceful shutdown on SIGTERM/SIGINT
- [x] T-39: `api/internal/ws/hub.go` — `Hub` with connections map[scanRunID]map[connID]chan[]byte, channel-based register/unregister/broadcast, `Run(ctx)` goroutine
- [x] T-40: `api/internal/ws/handler.go` — `ScanProgressHandler(hub, jwtSecret)` — JWT from ?token= query param, gorilla/websocket upgrade, pumps messages to client
- [x] T-41: `api/internal/ws/listener.go` — `StartListener(ctx, pool, hub)` — dedicated pgx conn, LISTEN scan_progress, parse NOTIFY JSON → hub.Broadcast, reconnects with exponential backoff
- [x] T-42: WS wiring in main.go — hub.Run() goroutine started, ws.StartListener() goroutine started, GET /ws/scan mounted
- [x] T-43: WS envelope verified in hub.go — Broadcast docs specify `{"event":"...","scan_run_id":"...","ts":"...","data":{}}`, listener passes raw NOTIFY payload without re-wrapping
- [x] T-44: `api/Dockerfile` — multi-stage: golang:1.23-alpine builder → gcr.io/distroless/static-debian12 final, CGO_ENABLED=0 GOOS=linux

### Commit
- 8b59446: `feat(api): M2+M4 Go API auth, config, middleware, JWT, OAuth2, WebSocket`

---

## PR-3: Milestone 3 — Go API Domain Handlers
_Completed: 2026-06-04_

### Completed
- [x] T-23: `api/internal/jobs/repo.go` — `Repo` struct wrapping sqlc Queries via `stdlib.OpenDBFromPool`; methods: `ListByUser`, `GetByID`, `Insert`, `UpdateStatus`, `UpsertByURL`
- [x] T-24: `api/internal/jobs/service.go` — `Service` with `AddManual` (https validation + platform detection from hostname), `List`, `GetByID` (returns ErrNotFound for wrong user); `detectPlatform` covers greenhouse/ashby/lever/recruitee/smartrecruiters/workable
- [x] T-25: `api/internal/jobs/handler.go` — `GET /api/jobs?page&limit` → `{jobs, page, limit}`; `POST /api/jobs {url}` → 201 `{id, url, status, platform}`; `GET /api/jobs/{id}` → 200 or 404; JSON error format `{error, code}`
- [x] T-26: Jobs wiring in `api/cmd/api/main.go` — `jobs.NewHandler(jobs.NewService(pool)).RegisterRoutes(r)` inside auth+tenant group
- [x] T-27: `api/internal/companies/service.go` — `DetectProvider(url) string`, `List`, `Add` (auto-detect provider if not supplied), `Remove` (ownership check + delete)
- [x] T-28: `api/internal/companies/handler.go` — `GET /api/companies`, `POST /api/companies {name, careers_url, provider_id?}` → 201, `DELETE /api/companies/{id}` → 204
- [x] T-29: Companies wiring in main.go
- [x] T-30: `api/internal/scan/service.go` — `TriggerScan`: list enabled companies → InsertScanRun → Enqueue "scan-company" per company; `GetScanRun` by ID
- [x] T-31: `api/internal/scan/handler.go` — `POST /api/scan` → 202 `{scan_run_id}`; `GET /api/scan-runs/{id}` → 200 `{id, status, new_jobs, errors, started_at, finished_at}`
- [x] T-32: Scan wiring in main.go
- [x] T-33: `api/internal/evaluate/service.go` — `EnqueueEvaluation`: job ownership check, usage limit check (free plan ≤5/month), Enqueue "evaluate-job"; `GetReport`: job→application→report chain
- [x] T-34: `api/internal/evaluate/handler.go` — `POST /api/jobs/{id}/evaluate` → 202; `GET /api/jobs/{id}/report` → 200 `{blocks_json, content_md}` or 404
- [x] T-35: Evaluate wiring in main.go
- [x] T-36: `api/internal/cv/service.go` — `EnqueuePDFGeneration`: checks application+report+master CV exist, Enqueue "generate-pdf"; `GetDownloadURL`: gets pdf_path → R2 signed URL (24h); `ListCVs`, `CreateCV`, `SetMasterCV`
- [x] T-37: `api/internal/cv/handler.go` — `POST /api/jobs/{id}/cv` → 202; `GET /api/jobs/{id}/cv` → 200 `{download_url, expires_at}` or 404; `GET /api/cvs`; `POST /api/cvs` → 201; CV wired in main.go with r2Client
- [x] T-38: `api/internal/tracker/service.go` + `handler.go` — `GET /api/applications?page&limit`; `PATCH /api/applications/{id} {status?, notes?}` → 200 or 400/404; ValidStatuses constant, ErrInvalidStatus; wired in main.go

### Commit
- b3047a1: `feat(api): M3 domain handlers — jobs, companies, scan, evaluate, cv, tracker`

---

## Pending milestones
M5, M6, M7, M8

---

## Notes
- T-07 migration inlines both schema.sql and rls.sql content directly rather than using `\i` includes, for Docker init compatibility
- `FORCE ROW LEVEL SECURITY` added to rls.sql (beyond design spec requirement of just `ENABLE`) to ensure app_user cannot bypass RLS
- T-10 was NOT auto-installed per the explicit instruction in the task description
- sqlc generated code uses `database/sql` interface — all domain services wrap pgxpool via `stdlib.OpenDBFromPool(pool)` to satisfy the DBTX interface
- `go build ./...` compiles cleanly with go1.25.0
- `min()` function implemented manually in listener.go (pre-go1.21 compatibility for distroless build with go1.23 image)
- Branch: `feat/1-career-ops-saas-mvp` in `/home/k3n5h1n/Escritorio/career-ops-saas`
- Commits: 03ec85d (T-01), 3452ed8 (T-02), dffd2e6 (T-03), dd7fafc (T-04–07), dbadc0c (T-08), a785e4f (T-09), 23327f6 (T-10 pending), a3a6a38 (T-10 sqlc generate done), 8b59446 (M2+M4), b3047a1 (M3)
- evaluate/service.go: free plan limit check uses ErrNoRows to detect missing usage row (evaluations_count = 0 in that case); pro/unlimited plan bypass is a TODO
- cv/handler.go: r2Client is nil-safe — returns 503 if R2 not configured, so local dev without R2 still works for other endpoints
