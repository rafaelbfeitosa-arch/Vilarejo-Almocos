# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Almoço Escolar · Vilarejo** — a mobile-first web app for managing school lunches at Escola Vilarejo. Parents confirm/cancel their children's daily meals; a cook enters the weekly menu; an admin views lists and exports monthly reports.

The project is split into two parts:
- **Frontend** — a single `index.html` (vanilla JS, no build step) hosted via GitHub Pages or static hosting
- **Backend** — a Flask app deployed on PythonAnywhere at `https://rafaelbf.pythonanywhere.com`. The source lives in its own **private** repo, `Vilarejo-Backend` (`C:\Users\rafae\Vilarejo-Backend\app.py`, ~2900 lines, serves both apps). It is private because it holds the admin routes and auth logic; the two frontend repos must stay public to be served by GitHub Pages.

## Architecture

### Frontend (`index.html`)
Single-page app with screen-based navigation (no router library). Key constants at the top:
- `API` — backend base URL
- `CODIGO_ESCOLA = 'VILAREJO'` — unlocks the admin lunch list view
- `CODIGO_COZINHA = 'COZINHA'` — unlocks the cook's weekly menu screen

Screens (`tela-*` divs) are shown/hidden via `irPara(id)`. State is held in module-level `let` variables (`criancaAtual`, `cardapioAtual`, etc.).

**Frontend design:** for any UI/visual work on `index.html` (new screens, layout, styling), use the `frontend-design` skill (installed in `.claude/skills/frontend-design`). Apply it within the existing Waldorf visual identity shared by Almoço, Tardes Brincantes and the parents' portal — don't replace the palette, fonts or tone with a new aesthetic. Keep it mobile-first and vanilla (no frameworks, no build step).

### Backend (`Vilarejo-Backend/app.py`, private repo)
Single Flask file serving two independent apps sharing one process:

| App | DB | Purpose |
|---|---|---|
| Almoço Escolar | `almoco.db` (SQLite) | Children, fixed days, confirmations, extra adult lunches, cardápio |
| Tardes Brincantes | `tardes_brincantes.db` (SQLite) | After-school program presence |

**Almoço DB tables:** `criancas`, `dias_fixos`, `confirmacoes`, `avulsos`, `cardapio`

**Key logic:** A child's lunch for a given day is computed by `status_dia()` — it layers fixed recurring days → exception records in `confirmacoes`. The `lista_do_dia()` helper assembles the daily list of **students** used by the API and the email/XLSX reports.

**`avulsos` is not the child's lunch.** Despite the name (it started as "quentinha for the child to take on a trip"), the table now holds **extra adult lunches ordered by the family and billed to the student's account** — `local='escola'` is a plate served at school, `local='casa'` is a pot to take home. They are separate portions: `status_dia()` deliberately ignores the table, and `extras_do_dia()` returns them as their own list. `/lista-dia` concatenates both, tagged by `categoria` (`aluno` | `adulto`). Never merge them back by `crianca_id` — that bug made a child with a fixed day plus two extra portions count as 2 instead of 3. See `backend/PLANO-almoco-pais.md`.

**Authentication:** Three secrets from `.env` — `LISTA_SECRET` (admin/lista access), `TB_SECRET` (Tardes Brincantes admin) and `PROF_SECRET` (teachers/admin: edit lunches with no time limit). The cook's code (`COZINHA`) is frontend-only and not validated by the backend.

None of these have a default in the code, and the checks fail **closed**: a missing env var denies access rather than matching the empty string. Never reintroduce a fallback value — it becomes a published password on the first commit.

**Edit deadline:** parents can change *today's* lunch only until 10:00 (America/Recife); future days stay free, past days are blocked. The single source of truth is `prazo_aberto()`, used by `/dias` and by every write endpoint. `PROF_SECRET`, sent as the `X-Prof-Secret` header, bypasses the cutoff. Changing `dias_fixos` after the cutoff would alter today through `status_dia()`'s layering, so `_congelar_hoje_se_fechado()` pins today with an explicit `confirmacoes` row. Any new header must be added to `Access-Control-Allow-Headers` — the app is cross-origin.

### GitHub Actions (`.github/workflows/enviar-lista.yml`)
Runs Mon–Fri at 13:05 UTC (10:05 Brasília, right after the parents' cutoff), calls `POST /enviar-lista` with `X-Secret` header. Requires repo secrets: `API_URL` and `LISTA_SECRET`.

## Deployment

- **Frontend:** push to `main` — GitHub Pages serves `index.html` directly
- **Backend:** upload/edit `app.py` on PythonAnywhere via their editor, then click **Reload** on the Web tab. No CI/CD. Deploy the backend *before* the frontend when a change spans both.

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
PROF_SECRET
```

## Day name conventions

The frontend and backend use `DIAS_PT = ["Segunda", "Terca", "Quarta", "Quinta", "Sexta"]` (index matches Python's `date.weekday()`). The cardápio dict uses these same keys. Do not mix with display labels like `"Terça"` (with accent) — the storage keys are accent-free.
