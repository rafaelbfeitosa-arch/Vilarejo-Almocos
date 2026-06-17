# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Almoço Escolar · Vilarejo** — a mobile-first web app for managing school lunches at Escola Vilarejo. Parents confirm/cancel their children's daily meals; a cook enters the weekly menu; an admin views lists and exports monthly reports.

The project is split into two parts:
- **Frontend** — a single `index.html` (vanilla JS, no build step) hosted via GitHub Pages or static hosting
- **Backend** — a Flask app (`backend/flask_app.py`) deployed on PythonAnywhere at `https://rafaelbf.pythonanywhere.com`

## Architecture

### Frontend (`index.html`)
Single-page app with screen-based navigation (no router library). Key constants at the top:
- `API` — backend base URL
- `CODIGO_ESCOLA = 'VILAREJO'` — unlocks the admin lunch list view
- `CODIGO_COZINHA = 'COZINHA'` — unlocks the cook's weekly menu screen

Screens (`tela-*` divs) are shown/hidden via `irPara(id)`. State is held in module-level `let` variables (`criancaAtual`, `cardapioAtual`, etc.).

### Backend (`backend/flask_app.py`)
Single Flask file serving two independent apps sharing one process:

| App | DB | Purpose |
|---|---|---|
| Almoço Escolar | `almoco.db` (SQLite) | Children, fixed days, confirmations, avulsos, cardápio |
| Tardes Brincantes | `tardes_brincantes.db` (SQLite) | After-school program presence |

**Almoço DB tables:** `criancas`, `dias_fixos`, `confirmacoes`, `avulsos`, `cardapio`

**Key logic:** A child's lunch for a given day is computed by `status_dia()` — it layers fixed recurring days → exception records in `confirmacoes` → one-off `avulsos`. The `lista_do_dia()` helper assembles the full daily list used by both the API and email/XLSX reports.

**Authentication:** Two secrets from `.env` — `LISTA_SECRET` (admin/lista access) and `TB_SECRET` (Tardes Brincantes admin). The cook's code (`COZINHA`) is frontend-only and not validated by the backend.

### GitHub Actions (`.github/workflows/enviar-lista.yml`)
Runs Mon–Fri at 11:01 UTC, calls `POST /enviar-lista` with `X-Secret` header. Requires repo secrets: `API_URL` and `LISTA_SECRET`.

## Deployment

- **Frontend:** push to `main` — GitHub Pages serves `index.html` directly
- **Backend:** edit `flask_app.py` on PythonAnywhere via their editor, then click **Reload** on the Web tab. No CI/CD.

## Backend `.env` variables

```
DB_PATH
TB_DB_PATH
GMAIL_USER
GMAIL_APP_PASSWORD
COZINHA_EMAIL
LISTA_SECRET
BACKUP_EMAIL
TB_SECRET
TB_VIEW_CODE
```

## Day name conventions

The frontend and backend use `DIAS_PT = ["Segunda", "Terca", "Quarta", "Quinta", "Sexta"]` (index matches Python's `date.weekday()`). The cardápio dict uses these same keys. Do not mix with display labels like `"Terça"` (with accent) — the storage keys are accent-free.
