# Mapa de Perfil — Miguel Angel Anay Gomez

Referencia rápida de dónde vive cada dato personal en career-ops.

---

## `cv.md` — Fuente de verdad del CV

Todo lo que aparece en el PDF generado viene de aquí.

| Sección | Contenido |
|---------|-----------|
| Header | Nombre, email, teléfono, LinkedIn, Portfolio, GitHub, CIP |
| Professional Summary | Narrative de 12+ años, stack, rol actual |
| Work Experience | Yiwu (Abr 2025–presente), Encora/Profuturo (Nov 2020–Abr 2025), Prosegur, Medios Industriales |
| Technical Skills | 12 categorías: Languages, Frontend, Backend, AI/MLOps, Cloud AWS, Cloud Azure, DevOps, Security, Databases, ERP, Mobile, Testing |
| Education | UNI (Ing. Industrial 2004–2012) + Diplomado UPN (Ene–Sep 2025) |
| Certifications | Advanced Data Engineer · Arquitectura Hexagonal · Python DSRP · Gen AI Coursera · AZ-900 · BI UNI · SQL Server UNI · VB.NET UNI |
| Key Projects | Agencia Virtual, Clave Web, FactuFácil, YUPI/Cajonero, preciodecajon.pe |
| Languages | Español nativo, Inglés intermedio |

---

## `config/profile.yml` — Configuración del sistema

Datos que el sistema usa para scoring, evaluaciones y negociación.

| Sección | Contenido |
|---------|-----------|
| `candidate` | Nombre, email, teléfono, LinkedIn, portfolio, GitHub, CIP |
| `target_roles` | 5 roles primarios + 6 arquetipos con nivel y fit |
| `narrative` | Headline, exit story, superpowers, proof points con URLs y hero metrics |
| `compensation` | Target $3,000–5,000 USD/mes · Mínimo $2,500 |
| `location` | Lima, UTC-5, disponibilidad on-site/remote |
| `languages` | Español nativo, Inglés intermedio |
| `education` | UNI + Diplomado UPN |
| `cv.output_format` | `html` → PDF via Playwright |

---

## `modes/_profile.md` — Arquetipos y scoring

Lo que el sistema lee para adaptar evaluaciones a tu perfil.

| Sección | Contenido |
|---------|-----------|
| Target Roles | Tabla de arquetipos con ejes temáticos y qué compran |
| Adaptive Framing | Qué enfatizar según tipo de rol |
| Exit Narrative | Historia de Profuturo → CTO/builder |
| Portfolio/Demo | FactuFácil, Agencia Virtual, Clave Web, preciodecajon.pe |
| Comp Targets | Rangos y scripts de negociación |
| Location Policy | Reglas de scoring por modalidad |
| Scoring Adjustments | Boost +0.3 / Penalizar -0.3 / Auto-SKIP |

---

## `portals.yml` — Scanner de ofertas

| Sección | Estado actual |
|---------|---------------|
| `title_filter.positive` | Keywords EN + ES para roles target |
| `title_filter.negative` | Excluye Junior, Intern, tecnologías ajenas |
| `search_queries` | 12 activos: Bumeran x4, Computrabajo x4, LinkedIn x4 |
| `tracked_companies` | 96 empresas EU/US (deshabilitadas — activar cuando busques mercado internacional) |

---

## Archivos a completar (pendientes)

| Archivo | Estado | Para qué sirve |
|---------|--------|----------------|
| `article-digest.md` | No existe | Proof points detallados de proyectos y artículos — enriquece CVs tailoreados |
| `interview-prep/story-bank.md` | Vacío | Historias STAR — se llena automáticamente al evaluar ofertas |
| `data/follow-ups.md` | No existe | Historial de seguimiento a aplicaciones |

---

## Regla de edición

| Si quieres cambiar... | Edita... |
|-----------------------|---------|
| Datos de contacto | `cv.md` + `config/profile.yml` |
| Experiencia laboral / proyectos | `cv.md` |
| Certificaciones | `cv.md` |
| Salario objetivo | `config/profile.yml` → `compensation` |
| Roles target | `config/profile.yml` → `target_roles` + `modes/_profile.md` |
| Scoring y arquetipos | `modes/_profile.md` |
| Portales de búsqueda | `portals.yml` |

---

*Última actualización: Mayo 2026*
