# Exploration: career-ops-saas-mvp

## Reusable Code Inventory

| Módulo | Qué hace | Para el worker SaaS |
|---|---|---|
| `providers/_http.mjs` | `fetchJson`, `fetchText`, `makeHttpCtx()` — timeout 10s, SSRF-safe | Reusable directo |
| `providers/_types.js` | JSDoc types: `Job`, `PortalEntry`, `Context` | Forma directa de la tabla `jobs` en PostgreSQL |
| `providers/greenhouse.mjs` | API boards-api.greenhouse.io, allowlist SSRF | Reusable sin cambios |
| `providers/ashby.mjs` | 30s timeout, 2-retry backoff+jitter (Ashby tiene 10s+ latency) | Reusable, crítico mantener el backoff |
| `providers/lever.mjs` | Simple, sin paginación | Reusable |
| `providers/recruitee.mjs` | Regex subdomain allowlist, exporta `parseRecruiteeResponse` | Reusable |
| `providers/smartrecruiters.mjs` | Paginado hasta 5000 postings, exporta `parseSmartRecruitersResponse` | Reusable |
| `providers/workable.mjs` | Feed `apply.workable.com/{slug}/jobs.md` no documentado | Reusable pero **FRÁGIL** — endpoint no oficial |
| `providers/local-parser.mjs` | `execFile` subprocess | **NO reusable** — incompatible con workers multi-tenant |
| `generate-pdf.mjs` | `normalizeTextForATS()` + generación HTML→PDF con Playwright | Extraer como export async, envolver como endpoint del worker |
| `scan.mjs` | Pattern de carga de providers + concurrencia=10 | Reusable logic, reemplazar file I/O por writes a PostgreSQL |
| `liveness-core.mjs` | 16 regex (EN/DE/FR) para detectar ofertas expiradas + `APPLY_PATTERNS` | Reusable directo como módulo compartido |

**6 de 7 providers listos para producción. Solo local-parser no aplica a SaaS.**

---

## Data Model → PostgreSQL

| CLI (archivos) | PostgreSQL |
|---|---|
| `data/applications.md` columnas | `applications (id, user_id, job_id, score, status, pdf_path, report_path, notes, created_at)` |
| `portals.yml > tracked_companies` | `watched_companies (id, user_id, name, careers_url, provider_id, enabled)` |
| `data/scan-history.tsv` (dedup por URL) | `jobs.url UNIQUE` — upsert, actualizar `last_seen` |
| `templates/states.yml` (8 estados) | Enum/check constraint en `applications.status` |
| `reports/*.md` | `reports (id, application_id, content_md, blocks_json, created_at)` |

Entidades nuevas: `users`, `subscriptions`, `scan_runs`, `gmail_connections`.

RLS: `user_id` en cada tabla con políticas PostgreSQL.

---

## Estructura del Prompt de Evaluación IA

El prompt en `modes/oferta.md` es un pipeline de 7 bloques:

1. **Step 0** — Detección de arquetipo (6 tipos: FDE, SA, PM, LLMOps, Agentic, Transformation)
2. **Bloque A** — Tabla resumen del rol
3. **Bloque B** — JD requirement → línea del CV (gaps con mitigación)
4. **Bloque C** — Alineación de nivel
5. **Bloque D** — Datos de mercado salarial via WebSearch
6. **Bloque E** — Diff de personalización del CV
7. **Bloque F** — Stories STAR
8. **Bloque G** — Score de legitimidad del posting

**Adaptaciones para server-side:**
- `cv.md` + `_profile.md` → `user.cv_markdown` + `user.profile_json` desde DB
- Block D WebSearch → Serper/Brave API o datos pre-cacheados
- Output (markdown) → parsear en `blocks_json` + `content_md`
- Prompt caching: cachear `[system prompt + CV del usuario]`, variar solo el JD

---

## Gaps — Lo que hay que construir desde cero

| Componente | Esfuerzo |
|---|---|
| Go API (chi, JWT middleware, todos los handlers) | Alto |
| Auth Google OAuth2 + JWT | Medio |
| Billing Stripe + usage metering | Medio |
| WebSocket scan progress streaming | Medio |
| Gmail OAuth2 + GCP Pub/Sub + parser de alertas | Alto |
| PostgreSQL schema multi-tenant + RLS + migrations | Medio |
| Worker HTTP API (Express + BullMQ/pg-boss) | Medio |
| Reescritura scan.mjs (file I/O → DB writes) | Medio |
| Frontend Next.js + shadcn/ui | Alto |
| R2 Storage signed URLs | Bajo |

---

## Riesgos

- **Workable feed** no documentado — puede romperse sin aviso
- **Ashby rate limits** — requiere throttling por usuario en la queue del worker
- **Gmail Pub/Sub** necesita infraestructura GCP — polling es más seguro para MVP
- **Playwright/Chromium** en Docker agrega ~500MB, necesita `--no-sandbox`; aislar worker de PDF del worker de scan
- **Multi-tenancy**: todos los query paths deben auditarse para evitar cross-tenant leaks con RLS
- **Prompt caching ROI** desconocido hasta medirlo en producción
- **Sin test runner** configurado aún (`strict_tdd: false`)

---

## Recomendación

Portear los 7 providers HTTP y `liveness-core` al worker Node.js (Express + BullMQ), reemplazando file I/O por PostgreSQL writes. Construir el Go API como capa delgada de auth + routing + WebSocket que encola jobs vía pg-boss/BullMQ. **Diferir Gmail y Stripe a MVP+1.** El loop central (scan → evaluate → generate PDF → track) es alcanzable con el código existente y entrega valor real antes de las integraciones complejas.
