# Proposal: career-ops-saas MVP (Fase 1)

## Intent

The career-ops CLI works only for one technical user on their own machine: file-based state, manual command runs, no auth, no isolation. To serve non-technical job seekers as a product, the same core loop — scan → evaluate → generate CV → track — must run as a multi-tenant web service. career-ops-saas delivers that loop as a hosted platform so any user signs in, manages job offers in a dashboard, and gets AI evaluations and tailored CVs without touching a terminal.

## Proposed Solution

A monorepo SaaS with three services. A **Go API** is the thin auth + routing + WebSocket layer. A **Node.js worker** reuses the existing `providers/*.mjs`, `liveness-core.mjs`, and `generate-pdf.mjs` to scan portals, run AI evaluations, and render PDFs — zero provider rewrite. **PostgreSQL** holds all state with row-level multi-tenancy. A **Next.js** frontend gives users a date-sorted job dashboard, detail panel, evaluate/generate-CV actions, and a status tracker. The Go API enqueues async work via pg-boss; the worker consumes it and streams scan progress back over WebSocket.

## Scope

### In Scope
- **Go API**: Google OAuth2 + JWT auth, job listings CRUD, scan trigger, evaluate trigger, CV/PDF endpoint, WebSocket for scan progress
- **Node.js worker**: scan (existing providers), evaluate (Anthropic API), PDF generation (existing `generate-pdf.mjs`)
- **PostgreSQL schema**: `users`, `jobs`, `watched_companies`, `applications`, `reports`, `cvs` with RLS
- **Next.js frontend**: dashboard (jobs sorted by date), job detail panel, evaluate action, generate-CV action, tracker
- pg-boss job queue (pure PostgreSQL, no Redis)

### Out of Scope (deferred)
- Gmail integration / Pub/Sub email parsing → MVP+1
- Stripe billing + usage metering → MVP+1
- Playwright apply flow + WebSocket screenshot streaming → Phase 2
- Turkish / French / German / Japanese language modes
- `local-parser.mjs` (subprocess model incompatible with multi-tenant workers)

## Capabilities

### New Capabilities
- `auth`: Google OAuth2 login, JWT issue/verify, per-user session
- `job-listings`: CRUD for jobs, manual URL add, date-sorted retrieval, RLS isolation
- `scan`: trigger portal scan via worker, upsert results, WebSocket progress
- `evaluation`: trigger AI evaluation (Anthropic), persist score + report blocks
- `cv-generation`: trigger tailored PDF generation, return signed download
- `tracker`: application status lifecycle (canonical states)
- `multi-tenancy`: PostgreSQL RLS policies across all user-scoped tables
- `job-queue`: pg-boss enqueue/consume between API and worker

### Modified Capabilities
None — this is a greenfield service; the CLI is reused as library code, not modified.

## Approach

Monorepo `career-ops-saas/` with `api/` (Go), `worker/` (Node.js), `web/` (Next.js). The worker wraps existing provider modules behind an HTTP/queue interface, replacing file I/O with PostgreSQL writes. Go owns auth, request routing, and the WebSocket hub; it never scrapes or calls Anthropic directly — it delegates to the worker via pg-boss. RLS enforces tenant isolation at the database layer so no query path can leak cross-tenant data. Prompt caching on `[system prompt + user CV]` keeps evaluation cost near $0.02/eval.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `career-ops-saas/api` | New | Go service: auth, handlers, WS hub |
| `career-ops-saas/worker` | New | Node worker reusing `providers/*.mjs`, `generate-pdf.mjs`, `liveness-core.mjs` |
| `career-ops-saas/web` | New | Next.js dashboard + detail + tracker |
| `career-ops-saas/db` | New | Schema, RLS policies, pg-boss tables, migrations |
| `providers/*.mjs` (existing) | Reused | Imported by worker, unchanged |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Workable feed (undocumented endpoint) breaks | Med | Isolate provider; degrade gracefully, alert on failure |
| Ashby rate limits under multi-user load | Med | Keep existing backoff; per-user throttle in worker queue |
| Playwright/Chromium container size (~500MB, `--no-sandbox`) | Med | Isolate PDF worker from scan worker; defer apply-flow Playwright to Phase 2 |
| RLS cross-tenant leak | High impact | Audit every query path; integration tests asserting tenant isolation |
| Prompt caching ROI unknown | Low | Measure in production; cost still acceptable without caching |

## Rollback Plan

Greenfield service in its own monorepo — no existing system to break. Per-component rollback: revert the service deployment to the previous image; PostgreSQL migrations are versioned and down-migratable. If MVP fails validation, the CLI remains fully functional and untouched.

## Dependencies

- Anthropic API key (evaluation)
- Google OAuth2 client credentials
- PostgreSQL 15+ (RLS + pg-boss)
- Object storage (signed URLs) for generated PDFs

## Success Criteria

- [ ] User can sign in with Google
- [ ] User can manually add a job URL and have it scanned
- [ ] User can evaluate an offer (AI score + report)
- [ ] User can generate a tailored CV PDF
- [ ] User can track application status
- [ ] System handles 50 concurrent users without degradation

## Pricing Fit

| Plan | Price | Evaluations | PDFs |
|------|-------|-------------|------|
| Free | $0 | 5 / month | — |
| Pro | $29 / month | 50 / month | ✅ |
| Unlimited | $59 / month | ∞ | ✅ |

Billing enforcement deferred to MVP+1; MVP ships the metering-ready schema (`usage` counters) without Stripe.
