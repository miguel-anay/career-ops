# Spec: career-ops-saas MVP (Fase 1)

_Change:_ career-ops-saas-mvp
_Status:_ draft
_Date:_ 2026-06-03

---

## Functional Requirements

**FR-01 — Google OAuth2 sign-in**
The system MUST accept Google OAuth2 as the sole authentication method. On first sign-in, a user record is created automatically. Subsequent sign-ins reuse the existing record matched by Google subject ID.

**FR-02 — JWT issuance and refresh**
After successful OAuth2 callback, the API MUST issue an access token (JWT, 1h expiry) and a refresh token (opaque, 7d expiry). Clients MUST use the refresh endpoint to rotate tokens without re-authenticating via Google.

**FR-03 — Manual job URL add**
An authenticated user MUST be able to submit a job posting URL. The API records it as a new `job` row owned by that user with status `pending` and enqueues a liveness check.

**FR-04 — Watched companies**
A user MUST be able to add and remove companies from their watched list. Each entry stores the company name, careers URL, and which ATS provider to use (greenhouse, ashby, lever, recruitee, smartrecruiters, workable).

**FR-05 — Scan trigger**
A user MUST be able to trigger a portal scan across all their watched companies. The API enqueues one pg-boss job per watched company. The worker executes the appropriate provider module, upserts discovered jobs into the `jobs` table for that user, and emits progress events over WebSocket.

**FR-06 — Job listings retrieval**
The API MUST return a paginated list of the requesting user's jobs, sorted by `received_at` descending. The response MUST include job title, company, URL, status, score (if evaluated), and `received_at`.

**FR-07 — AI evaluation**
A user MUST be able to trigger evaluation of a specific job. The API enqueues an evaluation job; the worker calls the Anthropic API using the user's stored CV and profile, parses the 7-block output, and persists `score` and `blocks_json` on the `reports` table row linked to that job.

**FR-08 — Evaluation report retrieval**
The API MUST return the full evaluation report (all blocks in structured JSON and raw markdown) for a job that has been evaluated. Requesting a report for an unevaluated job returns 404.

**FR-09 — CV PDF generation**
A user MUST be able to trigger PDF generation for a specific job that has been evaluated. The worker runs `generate-pdf.mjs` with tailored content, uploads the result to object storage, and returns a signed download URL valid for 24h.

**FR-10 — Application tracker**
A user MUST be able to view all their applications. Each application record exposes job info, score, current status, PDF availability, and notes.

**FR-11 — Application status update**
A user MUST be able to update the status and notes of any of their applications. Status values MUST be constrained to the canonical eight states: `Evaluated`, `Applied`, `Responded`, `Interview`, `Offer`, `Rejected`, `Discarded`, `SKIP`.

**FR-12 — WebSocket scan progress**
While a scan is running, the API MUST push progress events to the authenticated client's WebSocket connection. Events: `scan.started`, `scan.company.done` (with job count), `scan.company.error` (provider failure), `scan.completed` (totals).

---

## Non-Functional Requirements

**NFR-01 — Multi-tenancy isolation**
Every query touching `jobs`, `watched_companies`, `applications`, `reports`, or `cvs` MUST be filtered by `user_id` enforced at the PostgreSQL RLS layer. Application-layer filtering alone is not sufficient.

**NFR-02 — Scan performance**
A full scan across 20 watched companies MUST complete in under 60s (providers running concurrently, cap=10).

**NFR-03 — AI evaluation latency**
Evaluation MUST return a completed report in under 30s. Prompt caching MUST be applied to the `[system prompt + user CV]` prefix.

**NFR-04 — Availability**
99.5% monthly uptime target, excluding announced maintenance windows.

**NFR-05 — Security**
HTTPS only. JWTs verified on every request; expired tokens rejected with 401. Provider HTTP clients MUST use the SSRF allowlist from `providers/_http.mjs`. Refresh tokens rotated on each use.

**NFR-06 — Concurrent load**
50 concurrent users at MVP; p95 API latency ≤ 500ms for non-async endpoints.

**NFR-07 — Graceful provider failure**
If a provider throws or times out, the scan continues for remaining companies. Failed company emits `scan.company.error`. Scan is NOT marked failed unless all companies fail.

**NFR-08 — Metering-ready schema**
Schema MUST include `usage` counters per user per month (evaluations, PDFs) to support billing enforcement in MVP+1.

---

## API Contract

All endpoints require `Authorization: Bearer <access_token>` unless noted.

| Method | Path | Request | Response |
|--------|------|---------|----------|
| GET | `/auth/google` | — | 302 → Google consent |
| GET | `/auth/google/callback` | `?code=&state=` | 200 `{ access_token, refresh_token }` |
| POST | `/auth/refresh` | `{ refresh_token }` | 200 `{ access_token, refresh_token }` |
| GET | `/api/jobs` | `?page=&limit=` | 200 `{ jobs[], total, page }` |
| POST | `/api/jobs` | `{ url }` | 201 `{ id, url, status }` |
| GET | `/api/jobs/:id` | — | 200 `Job` or 404 |
| POST | `/api/jobs/:id/evaluate` | — | 202 `{ job_id, queue_id }` |
| GET | `/api/jobs/:id/report` | — | 200 `{ blocks_json, content_md }` or 404 |
| POST | `/api/jobs/:id/cv` | — | 202 `{ job_id, queue_id }` |
| GET | `/api/jobs/:id/cv` | — | 200 `{ download_url, expires_at }` or 404 |
| GET | `/api/applications` | `?page=&limit=` | 200 `{ applications[], total }` |
| PATCH | `/api/applications/:id` | `{ status?, notes? }` | 200 `Application` |
| GET | `/api/companies` | — | 200 `{ companies[] }` |
| POST | `/api/companies` | `{ name, careers_url, provider_id }` | 201 `WatchedCompany` |
| DELETE | `/api/companies/:id` | — | 204 |
| POST | `/api/scan` | — | 202 `{ scan_run_id }` |
| WS | `/ws/scan` | `?token=<access_token>` | Push: `ScanEvent` |

**Error shape:** `{ error: string, code: string }` — 400 validation, 401 unauthenticated, 403 forbidden, 404 not found, 409 conflict, 500 internal.

---

## Scenarios

**SC-01 — First-time Google sign-in**
Given: a new visitor with no existing account.
When: they complete the Google OAuth2 consent flow.
Then: a `users` row is created; access + refresh tokens returned; dashboard loads with empty jobs list.

**SC-02 — Manual job URL add**
Given: an authenticated user.
When: they POST `/api/jobs` with a valid HTTPS URL.
Then: a `jobs` row is created with `status = "pending"` scoped to that user; 201 returned; job appears at top of dashboard.

**SC-03 — Scan finds new jobs at a watched company**
Given: a user has one watched company with `provider_id = "greenhouse"`.
When: they POST `/api/scan`.
Then: worker calls Greenhouse provider; new jobs upserted into `jobs` for that user; `scan.company.done` WebSocket event pushed; `scan.completed` closes the stream.

**SC-04 — Evaluation completes and report is displayed**
Given: a job row exists and CV/profile are stored.
When: the user triggers POST `/api/jobs/:id/evaluate` and the worker finishes.
Then: a `reports` row is created with `blocks_json` (7 blocks) and `score` in [0,5]; GET `/api/jobs/:id/report` returns 200.

**SC-05 — CV PDF generated and downloaded**
Given: a job that has been evaluated.
When: the user triggers POST `/api/jobs/:id/cv` and the worker finishes.
Then: PDF stored in object storage; GET `/api/jobs/:id/cv` returns signed URL valid 24h; tracker shows PDF available.

**SC-06 — Cross-tenant isolation**
Given: user A owns job id=X; user B holds a valid JWT.
When: user B requests GET `/api/jobs/X`.
Then: response is 403 (or 404); no data belonging to user A is returned, enforced by PostgreSQL RLS.

**SC-07 — Workable provider fails gracefully**
Given: user watches two companies — Greenhouse and Workable; Workable's feed returns 404.
When: the scan runs.
Then: Greenhouse jobs upserted successfully; `scan.company.error` emitted for Workable; scan completes with `partial` status; Greenhouse results are not discarded.

**SC-08 — Expired access token rejected**
Given: a client holds an access token older than 1h.
When: the client calls any authenticated endpoint.
Then: API returns 401 `{ error: "token expired", code: "AUTH_EXPIRED" }`; client must POST `/auth/refresh`.
