# ONBOARDING.md

## Stack

- **Runtime**: Node.js ESM (`*.mjs`) — v18+
- **Dependencies**: Playwright, js-yaml, dotenv, @google/generative-ai
- **Planned SaaS stack**: Go + chi (API), Node.js + Express + Playwright (worker), Next.js + shadcn/ui (web), PostgreSQL + sqlc, Cloudflare R2, Anthropic API, Stripe

## Estructura del proyecto

```
career-ops/
  data/           — tracker, pipeline, scan history (markdown/TSV)
  config/         — profile.yml (user layer, nunca auto-actualizado)
  modes/          — prompts del sistema (system layer)
  providers/      — ATS API providers (greenhouse, ashby, lever, etc.)
  templates/      — CV templates HTML/LaTeX, states.yml
  reports/        — evaluation reports por aplicación
  output/         — PDFs generados (gitignored)
  openspec/       — artefactos SDD (explore, proposal, spec, design, tasks)
  batch/          — batch processing scripts
  interview-prep/ — STAR stories, intel reports
  dashboard/      — TUI Go (main.go)
  .atl/           — skill-registry.md
```

## Flujo de trabajo (Trunk-Based Development)

- **Feature/cambio grande**: explore → propose → `/tbd` → sdd-ff → apply → verify → `/tbd`
- **Bug/cambio pequeño**: `/tbd "descripción"` → fix → `/tbd`
- Las ramas viven **1-2 días máximo**
- Todo mergea directo a `main` — no existe rama `develop`

## Branch strategy

- `main`: única rama permanente, siempre deployable
- `feat/N-nombre`: features con SDD (vida corta)
- `fix/N-nombre`: bugs y cambios pequeños (vida corta)

## Comandos frecuentes

```bash
node scan.mjs                  # escanear portales (zero-token)
node generate-pdf.mjs          # generar PDF del CV
node verify-pipeline.mjs       # health check del pipeline
node merge-tracker.mjs         # mergear tracker additions
node update-system.mjs check   # verificar actualizaciones
node update-system.mjs apply   # aplicar actualización
```

## Variables de entorno

Ver `.env.example` si existe. Variables clave:
- `ANTHROPIC_API_KEY` — para evaluaciones IA
- `GOOGLE_API_KEY` — para Gemini (opcional)

## Cambios completados

<!-- Se actualiza al cerrar cada issue -->
