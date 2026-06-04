# Design: career-ops-saas MVP (Fase 1)

## Architecture Overview

career-ops-saas is a **three-service monorepo** backed by a single PostgreSQL instance.

```
career-ops-saas/
├── api/                  # Go + chi — auth, routing, WebSocket hub, enqueues work
├── worker/               # Node.js + Express + Playwright — scan, evaluate, PDF
├── web/                  # Next.js + shadcn/ui — dashboard, detail, tracker
├── db/                   # SQL migrations, RLS policies, schema.sql, sqlc queries
├── docker-compose.yml
├── .env.example
└── README.md
```

**Responsibility split:**
- **Go API** — thin control plane: auth, routing, WebSocket hub, RLS tenant variable, enqueues work. NEVER scrapes portals, NEVER calls Anthropic, NEVER launches Chromium.
- **Node.js worker** — data plane: consumes pg-boss jobs, runs providers, calls Anthropic, renders PDFs with Playwright.
- **Next.js web** — presentation layer: talks only to Go API over HTTP + WebSocket.

**Core loop data flow:**
```
web ──POST /scan──> api ──INSERT scan_run + pg-boss send──> PostgreSQL
                                          worker BOSS.work ◄─┘
                                            │ providers/*.mjs hit ATS APIs
                                            │ UPSERT jobs
                                            └── NOTIFY ──> api WS hub ──> web

web ──POST /evaluate──> api ──pg-boss "evaluate-job"──> worker
                                            │ Anthropic API (cached prompt)
                                            └── INSERT applications + reports

web ──POST /cv──> api ──pg-boss "generate-pdf"──> worker
                                            │ generate-pdf.mjs + Playwright
                                            └── upload R2 → UPDATE pdf_path
```

Worker → API WebSocket fan-out uses PostgreSQL `LISTEN/NOTIFY` — no Redis needed.

---

## Go API — Directory Structure

```
api/
├── cmd/api/main.go
├── internal/
│   ├── config/config.go           # env: DATABASE_URL, JWT_SECRET, GOOGLE_*, R2_*
│   ├── auth/
│   │   ├── oauth.go               # Google OAuth2 code exchange, userinfo
│   │   ├── jwt.go                 # issue/verify HS256, claims (user_id, plan, exp)
│   │   ├── handler.go             # GET /auth/google, GET /auth/google/callback, POST /auth/refresh
│   │   └── service.go             # upsert user by google_id
│   ├── jobs/
│   │   ├── handler.go             # GET /api/jobs, GET /api/jobs/:id, POST /api/jobs
│   │   ├── service.go             # manual URL add, platform detection
│   │   └── repo.go                # sqlc-backed queries
│   ├── scan/
│   │   ├── handler.go             # POST /api/scan
│   │   └── service.go             # create scan_run, fan-out pg-boss "scan-company" jobs
│   ├── evaluate/
│   │   ├── handler.go             # POST /api/jobs/:id/evaluate
│   │   └── service.go             # usage check (free plan cap), enqueue "evaluate-job"
│   ├── cv/
│   │   ├── handler.go             # GET/POST /api/cvs, POST /api/jobs/:id/cv, GET /api/applications/:id/pdf
│   │   └── service.go             # enqueue "generate-pdf", signed R2 download URL
│   ├── tracker/
│   │   ├── handler.go             # GET /api/applications, PATCH /api/applications/:id
│   │   └── service.go             # canonical-state transition validation
│   ├── companies/
│   │   ├── handler.go             # GET/POST/DELETE /api/companies
│   │   └── service.go             # provider_id resolution from careers_url
│   ├── ws/
│   │   ├── hub.go                 # connection registry keyed by (user_id, scan_run_id)
│   │   ├── handler.go             # GET /ws/scan/:scan_run_id (upgrade, JWT auth)
│   │   └── listener.go            # pg LISTEN scan_progress → fan-out to hub
│   ├── middleware/
│   │   ├── auth.go                # bearer JWT validation → user_id into ctx
│   │   ├── tenant.go              # SET LOCAL app.current_user_id (RLS)
│   │   ├── logging.go
│   │   ├── recover.go
│   │   └── cors.go
│   ├── queue/boss.go              # pg-boss-compatible INSERT into pgboss.job
│   └── platform/
│       ├── postgres.go            # pgxpool init, tenant-scoped conn helper
│       └── r2.go                  # S3-compatible signed URL generation
├── db/                            # symlink to /db
├── sqlc.yaml
├── go.mod
└── Dockerfile
```

---

## Node.js Worker — Directory Structure

```
worker/
├── index.mjs                      # boot: pg-boss client, register workers, Express health
├── jobs/
│   ├── scan.mjs                   # "scan-company": load provider, fetch, UPSERT, NOTIFY
│   ├── evaluate.mjs               # "evaluate-job": build prompt, Anthropic, parse blocks, INSERT
│   └── pdf.mjs                    # "generate-pdf": Playwright render, R2 upload, UPDATE
├── providers/                     # COPIED verbatim from career-ops — unchanged
│   ├── _http.mjs
│   ├── _types.js
│   ├── greenhouse.mjs
│   ├── ashby.mjs
│   ├── lever.mjs
│   ├── recruitee.mjs
│   ├── smartrecruiters.mjs
│   └── workable.mjs               # FRAGILE: undocumented feed, graceful-degrade
├── shared/
│   ├── liveness-core.mjs          # COPIED — expired-offer detection
│   └── generate-pdf.mjs           # COPIED, refactored as async export
├── lib/
│   ├── db.mjs                     # pg Pool, tenant-scoped query helper
│   ├── queue.mjs                  # pg-boss instance, .work() registration
│   ├── anthropic.mjs              # Anthropic SDK, prompt-cache headers, retry/timeout
│   ├── r2.mjs                     # S3-compatible R2 upload client
│   ├── prompt.mjs                 # builds evaluation prompt from template + DB data
│   └── progress.mjs               # pg NOTIFY helper for scan progress
├── modes/oferta.template.md       # adapted system-prompt (CV/profile injected from DB)
├── package.json
└── Dockerfile                     # node:20 + Playwright Chromium, --no-sandbox
```

**`local-parser.mjs` intentionally absent** — `execFile` subprocess is incompatible with multi-tenant workers.

---

## PostgreSQL Schema — Full DDL

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Enums
CREATE TYPE plan_t        AS ENUM ('free', 'pro', 'unlimited');
CREATE TYPE job_status_t  AS ENUM ('new', 'scanned', 'evaluated', 'archived');
CREATE TYPE app_status_t  AS ENUM (
  'Evaluated','Applied','Responded','Interview','Offer',
  'Rejected','Discarded','SKIP'
);
CREATE TYPE scan_status_t AS ENUM ('running','completed','partial','failed');

-- users
CREATE TABLE users (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email        text NOT NULL UNIQUE,
  google_id    text NOT NULL UNIQUE,
  plan         plan_t NOT NULL DEFAULT 'free',
  cv_markdown  text,
  profile_json jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at   timestamptz NOT NULL DEFAULT now()
);

-- watched_companies
CREATE TABLE watched_companies (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        text NOT NULL,
  careers_url text,
  provider_id text,
  ats_api_url text,
  enabled     boolean NOT NULL DEFAULT true,
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_watched_companies_user ON watched_companies(user_id);

-- jobs
CREATE TABLE jobs (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title           text NOT NULL,
  company         text NOT NULL,
  url             text NOT NULL,
  platform        text,
  status          job_status_t NOT NULL DEFAULT 'new',
  scraped_content text,
  evaluation_json jsonb,
  received_at     timestamptz,
  created_at      timestamptz NOT NULL DEFAULT now(),
  UNIQUE (user_id, url)
);
CREATE INDEX idx_jobs_user_received ON jobs(user_id, received_at DESC);

-- applications
CREATE TABLE applications (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  job_id     uuid NOT NULL UNIQUE REFERENCES jobs(id) ON DELETE CASCADE,
  score      double precision,
  status     app_status_t NOT NULL DEFAULT 'Evaluated',
  notes      text,
  pdf_path   text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_applications_user ON applications(user_id);

-- reports
CREATE TABLE reports (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  application_id uuid NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
  content_md     text NOT NULL,
  blocks_json    jsonb NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_reports_application ON reports(application_id);

-- cvs
CREATE TABLE cvs (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title      text NOT NULL,
  content_md text NOT NULL,
  is_master  boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_cvs_user ON cvs(user_id);
CREATE UNIQUE INDEX uq_cvs_master ON cvs(user_id) WHERE is_master;

-- scan_runs
CREATE TABLE scan_runs (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  started_at  timestamptz NOT NULL DEFAULT now(),
  finished_at timestamptz,
  new_jobs    integer NOT NULL DEFAULT 0,
  errors_json jsonb NOT NULL DEFAULT '[]'::jsonb,
  status      scan_status_t NOT NULL DEFAULT 'running'
);
CREATE INDEX idx_scan_runs_user ON scan_runs(user_id, started_at DESC);

-- usage (metering-ready, billing deferred to MVP+1)
CREATE TABLE usage (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  month             char(7) NOT NULL,
  evaluations_count integer NOT NULL DEFAULT 0,
  pdfs_count        integer NOT NULL DEFAULT 0,
  UNIQUE (user_id, month)
);

-- Row-Level Security
ALTER TABLE users             ENABLE ROW LEVEL SECURITY;
ALTER TABLE watched_companies ENABLE ROW LEVEL SECURITY;
ALTER TABLE jobs              ENABLE ROW LEVEL SECURITY;
ALTER TABLE applications      ENABLE ROW LEVEL SECURITY;
ALTER TABLE reports           ENABLE ROW LEVEL SECURITY;
ALTER TABLE cvs               ENABLE ROW LEVEL SECURITY;
ALTER TABLE scan_runs         ENABLE ROW LEVEL SECURITY;
ALTER TABLE usage             ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_users ON users
  USING (id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (id = current_setting('app.current_user_id', true)::uuid);

CREATE POLICY tenant_watched_companies ON watched_companies
  USING (user_id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (user_id = current_setting('app.current_user_id', true)::uuid);

CREATE POLICY tenant_jobs ON jobs
  USING (user_id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (user_id = current_setting('app.current_user_id', true)::uuid);

CREATE POLICY tenant_applications ON applications
  USING (user_id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (user_id = current_setting('app.current_user_id', true)::uuid);

CREATE POLICY tenant_reports ON reports
  USING (user_id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (user_id = current_setting('app.current_user_id', true)::uuid);

CREATE POLICY tenant_cvs ON cvs
  USING (user_id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (user_id = current_setting('app.current_user_id', true)::uuid);

CREATE POLICY tenant_scan_runs ON scan_runs
  USING (user_id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (user_id = current_setting('app.current_user_id', true)::uuid);

CREATE POLICY tenant_usage ON usage
  USING (user_id = current_setting('app.current_user_id', true)::uuid)
  WITH CHECK (user_id = current_setting('app.current_user_id', true)::uuid);
```

**RLS notas críticas:**
- El runtime DB role NO debe ser el owner de las tablas (los owners bypassean RLS). Usar `app_user` dedicado con `FORCE ROW LEVEL SECURITY`.
- `reports` y `applications` llevan `user_id` redundante para que RLS filtre sin JOIN — denormalización intencional.
- El flujo OAuth (user upsert) corre sin tenant variable — usa una función `SECURITY DEFINER auth_upsert_user(email, google_id)`.

---

## pg-boss Job Queue

**Job types:**

```jsonc
// scan-company — uno por empresa watched en cada scan run
{ "user_id": "uuid", "company_id": "uuid", "scan_run_id": "uuid" }

// evaluate-job
{ "user_id": "uuid", "job_id": "uuid" }

// generate-pdf
{ "user_id": "uuid", "job_id": "uuid", "application_id": "uuid" }
```

**Convenciones:**
- Cada handler setea el tenant: `SET CONFIG app.current_user_id = $user_id` antes de cualquier query.
- `scan-company`: throttle por usuario vía pg-boss `teamSize` para mitigar rate limits de Ashby.
- `generate-pdf` corre en container separado (aísla Chromium ~500MB).
- Retries: `retryLimit: 2`, `retryBackoff: true`. Workable falla con graceful degrade → `scan_runs.errors_json`, scan termina `partial`.

---

## WebSocket Protocol

Envelope: `{ "event": string, "scan_run_id": uuid, "ts": ISO8601, "data": {} }`

```jsonc
{ "event": "scan.started",       "data": { "total_companies": 12 } }
{ "event": "scan.job_found",     "data": { "job_id", "title", "company", "url", "is_new": true } }
{ "event": "scan.company.done",  "data": { "company_id", "company", "found": 7, "new": 3 } }
{ "event": "scan.company.error", "data": { "company_id", "company", "provider", "error": "..." } }
{ "event": "scan.completed",     "data": { "status": "partial", "new_jobs": 9, "companies_ok": 11, "companies_failed": 1 } }
```

Fan-out vía PostgreSQL `LISTEN/NOTIFY` en canal `scan_progress`. Si el socket cae mid-scan, el cliente hace polling a `GET /api/scan-runs/:id`.

---

## Anthropic API Integration

- **Model:** `claude-sonnet-4-6`
- **max_tokens:** `8000`
- **temperature:** `0.2`
- **timeout:** `120s`

**Estructura de prompt (3 bloques con caching):**

```
system: [
  { type: "text", text: <static system prompt — modes/oferta.template.md>,
    cache_control: { type: "ephemeral" } },        // CACHE 1 — varía nunca
  { type: "text", text: <user CV + profile_json>,
    cache_control: { type: "ephemeral" } }         // CACHE 2 — varía por usuario
]
messages: [
  { role: "user", content: <JD scrapeada + output contract> }   // varía por oferta
]
```

Costo efectivo con caching: ~$0.02/evaluación. Block D (salary data) usa Serper/Brave API en vez de WebSearch in-model. Output parseado a `blocks_json` (A–G) + `content_md`. Si falla el parse, se persiste `blocks_json: {"parse_error": true}` — ninguna evaluación se pierde.

---

## Docker Compose (local dev)

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: careerops
      POSTGRES_PASSWORD: careerops
      POSTGRES_DB: careerops
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U careerops"]
      interval: 5s
      retries: 10

  api:
    build: ./api
    depends_on: { postgres: { condition: service_healthy } }
    environment:
      DATABASE_URL: postgres://app_user:app_pw@postgres:5432/careerops?sslmode=disable
      JWT_SECRET: dev-secret-change-me
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      GOOGLE_REDIRECT_URL: http://localhost:8080/auth/google/callback
      R2_ACCOUNT_ID: ${R2_ACCOUNT_ID}
      R2_ACCESS_KEY_ID: ${R2_ACCESS_KEY_ID}
      R2_SECRET_ACCESS_KEY: ${R2_SECRET_ACCESS_KEY}
      R2_BUCKET: careerops-pdfs
      WEB_ORIGIN: http://localhost:3000
    ports: ["8080:8080"]

  worker:
    build: ./worker
    depends_on: { postgres: { condition: service_healthy } }
    environment:
      DATABASE_URL: postgres://app_user:app_pw@postgres:5432/careerops?sslmode=disable
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      ANTHROPIC_MODEL: claude-sonnet-4-6
      R2_ACCOUNT_ID: ${R2_ACCOUNT_ID}
      R2_ACCESS_KEY_ID: ${R2_ACCESS_KEY_ID}
      R2_SECRET_ACCESS_KEY: ${R2_SECRET_ACCESS_KEY}
      R2_BUCKET: careerops-pdfs
      SERPER_API_KEY: ${SERPER_API_KEY}
    ports: ["3001:3001"]
    shm_size: "1gb"

  web:
    build: ./web
    depends_on: [api]
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8080
      NEXT_PUBLIC_WS_URL: ws://localhost:8080
    ports: ["3000:3000"]

volumes:
  pgdata:
```

En producción, `generate-pdf` corre como container separado (`WORKER_ROLE=pdf`) para aislar Chromium.

---

## Key Architectural Decisions

| # | Decisión | Alternativa rechazada | Razón |
|---|---|---|---|
| ADR-1 | pg-boss sobre Redis/BullMQ | Redis + BullMQ | PostgreSQL ya en stack; enqueue transaccional; sin broker extra al MVP |
| ADR-2 | Node.js worker retiene providers | Reescribir en Go | 6 providers production-ready con lógica específica ya probada; Playwright sin equivalente Go maduro |
| ADR-3 | RLS sobre filtrado en app layer | WHERE user_id = ? en cada query | RLS es invariante de DB; un WHERE olvidado = data breach; choke point único |
| ADR-4 | Monorepo al MVP | Polyrepo | Schema compartido; un PR cambia schema + Go queries + worker atomicamente |
| ADR-5 | Cloudflare R2 sobre AWS S3 | AWS S3 | Zero egress fees; API S3-compatible; menor burn rate al MVP |
| ADR-6 | UUID PKs (gen_random_uuid) | serial/bigserial | Ineguesable; no filtra info de conteo; minteable en worker antes de INSERT |
