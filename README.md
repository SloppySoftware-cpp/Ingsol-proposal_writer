# INGSOL Proposal Web App

Web version of the `ingsol-proposal-writer` skill. Claude drafts proposal content as
structured JSON. A person edits and reviews it in the browser, a reviewer approves an
exact version, and only then does the server render the final four-part PDF with the
skill's own engine. The app never runs code that Claude writes.

> **Not production-ready.** Phase 3 uses development sign-in with test users. Do not
> deploy it or put real client data in it until Phase 4 Zoho sign-in is in place.

## Status

| Phase | Scope | Status |
|---|---|---|
| 1 | Backend core: JSON schema, builder, gate/preview/render API | Done |
| 2 | AI drafting: guidelines as system prompt, Claude returns proposal JSON | Done (live Claude run still pending an API key) |
| 3 | React frontend, review editor, approval workflow, database, audit log | Done (development auth) |
| 4 | Zoho sign-in (accounts.zoho.in), Zoho CRM deal and quote lookup | Next |
| 5 | Deployment, PostgreSQL, monitoring | Planned |

## Architecture

```
Browser (React, Vite)                 FastAPI backend                         External
┌──────────────────────────┐  /api   ┌───────────────────────────────────┐
│ Dashboard                │ ──────▶ │ auth.py      session cookie, roles │
│ New proposal wizard      │ cookie  │ proposals.py records, versions,    │
│ Review editor (3 panels) │         │              approvals, audit      │
│ Approval dialog          │         │ drafting.py  Claude, JSON schema ──┼──▶ Claude API
│ PDF preview (blob)       │ ◀────── │ builder.py   JSON -> engine story  │
└──────────────────────────┘         │ engine/      skill render + gate   │
                                     │ SQLite/PostgreSQL via SQLAlchemy   │
                                     └───────────────────────────────────┘
```

- **No secrets in the browser.** The browser only calls `/api` on its own origin. The
  Anthropic key stays on the server, and the session cookie is HttpOnly and SameSite=Lax.
- **The server enforces everything.** Approval, version matching, the Final Gate and download
  permission are all checked by the backend. The UI only reflects them.
- **Uploads are private.** No route serves uploaded files. Files are referenced by opaque
  IDs, and only image IDs can be placed in a proposal.

## Run locally (VS Code)

Prerequisites: Python 3.11+, [uv](https://docs.astral.sh/uv/), Node 20+.

Terminal 1, backend:

```bash
cd backend
uv venv .venv && uv pip install -p .venv/bin/python -r requirements.txt
export ANTHROPIC_API_KEY=sk-ant-...           # only needed for AI drafting
.venv/bin/uvicorn app.main:app --reload       # http://127.0.0.1:8000 (API docs at /docs)
```

Terminal 2, frontend:

```bash
cd frontend
npm install
npm run dev                                   # http://localhost:5173
```

Open http://localhost:5173 and sign in with one of the test users. The database is
created and migrated automatically when the backend starts.

### Environment variables

Backend. Set these in the environment or in `backend/.env`, which is loaded at startup and git-ignored. Real environment variables win, and empty values count as unset:

| Variable | Default | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | none | Claude API key for drafting. Without it, drafting returns 503 and everything else works |
| `ANTHROPIC_WORKSPACE_ID` | none | Only for an API key not scoped to a workspace; sent as the `anthropic-workspace-id` header |
| `DATABASE_URL` | `sqlite:///backend/data/proposals.db` | SQLAlchemy URL. Use `postgresql+psycopg://...` for PostgreSQL (add the `psycopg` package) |
| `PROPOSAL_DATA_DIR` | `backend/data` | Uploads and the SQLite file |
| `AUTH_MODE` | `dev` | `dev` enables the test users. Any other value disables them |
| `SESSION_SECRET` | random per process in dev | Required outside dev |
| `SESSION_MAX_AGE` | `28800` | Session length in seconds |
| `COOKIE_SECURE` | `0` | Set to `1` behind HTTPS |
| `PROPOSAL_MODEL`, `PROPOSAL_DRAFT_EFFORT` | `claude-opus-5-5`, `high` | Drafting model and effort |

Frontend (see `frontend/.env.example`): `VITE_API_PROXY_TARGET` (default
`http://127.0.0.1:8000`), which is where the dev server proxies `/api`. Never put
secrets in `VITE_*` variables, because they are bundled into the browser code.

### Database and migrations

Alembic manages the schema (`backend/migrations`). The backend runs
`alembic upgrade head` on startup. To change the schema:

```bash
cd backend
.venv/bin/alembic revision --autogenerate -m "describe the change"
.venv/bin/alembic upgrade head
```

Tables: `proposals`, `proposal_versions` (one immutable snapshot plus SHA-256 per saved
version), `approvals`, `audit_logs`.

## Generating and reviewing a proposal

1. **New proposal.** Enter the client name, Deal ID and quote number by hand (Zoho lookup
   arrives in Phase 4). For the Deal ID and quote number, choose *Enter value*,
   *Not known yet* (prints [TO BE CONFIRMED] and blocks approval) or *None* (left off the
   cover).
2. **Inputs.** Upload PDF, DOCX, TXT or MD notes plus PNG, JPG or SVG diagrams and photos, and/or
   paste notes. *Analyse inputs* has Claude pre-fill the intake form and list missing
   mandatory items.
3. **Intake.** Confirm the entity, currency, contacts (order sets precedence), payment terms,
   validity, AMC and optional sections.
4. **Generate.** Claude drafts the four Parts. The draft is saved as v1 and the review screen
   opens. If Claude is unavailable, nothing is saved and the error is shown with a retry.
5. **Review editor.**
   - The left panel shows the outline, with red counts for unresolved items.
   - The centre panel edits the cover fields and every block type: paragraphs, bullets,
     tables (group, subtotal and total rows), Gantt charts, TBC items, images and photo grids.
   - The right panel shows the Final Gate (BLOCK, WARN, INFO), unresolved
     [TO BE CONFIRMED] items, open questions, reviewer notes, internal context the draft
     left out, and approval readiness.
   - *Save draft* creates a new version. *Preview PDF* renders a watermarked DRAFT of the
     saved version.
6. **Submit for review**, then a reviewer approves or returns it.

## Approval and download rules

- **States:** Draft → In review → Approved. A reviewer can return a proposal to Draft at
  any time, with a note.
- **Who may approve:** only users with the reviewer role, and never for a proposal they
  created.
- **Approval conditions:** approval applies to the exact current version. The server
  refuses it if:
  - the reviewer saw an older version
  - the Final Gate reports any BLOCK on that version's snapshot
  - any WARN is unacknowledged
  - the reviewer has not confirmed: *"I have verified that the Zoho CRM quotation matches
    the commercial details in Part 3."*
- **What is recorded:** the reviewer, the timestamp, the approved version, the content hash,
  the CRM confirmation and the acknowledged warnings.
- **Edits after approval:** any saved change creates a new version, invalidates the
  approval (recorded in the audit log) and returns the proposal to In review.
- **Final PDF:** `GET /api/proposals/{id}/pdf` renders only from the approved snapshot, and
  only while that approval is valid and the content hash still matches. Each download is
  audited.
- **Legacy `/api/render`:** this Phase 1 endpoint now returns 403 unless the submitted JSON
  is exactly the content of a valid approval.
- **Previews:** draft previews are always watermarked DRAFT. They are fetched with the
  session cookie and shown as local blob URLs, never shareable links.

## API

All endpoints except `/api/health` and `/api/auth/config` require a signed-in user.

| Endpoint | Purpose |
|---|---|
| `GET /api/auth/config`, `POST /api/auth/dev-login`, `POST /api/auth/logout`, `GET /api/auth/me` | Session (dev adapter) |
| `GET /api/roster?entity=` | Contacts for an entity |
| `POST /api/upload` | Source notes (PDF, DOCX, TXT, MD) or figures (PNG, JPG, SVG) |
| `POST /api/intake/analyze` | Claude pre-fills the intake form |
| `POST /api/draft` | Claude draft without saving (Phase 2) |
| `POST /api/gate`, `POST /api/preview` | Gate and DRAFT preview for posted JSON (Phase 1) |
| `POST /api/render` | Final PDF for posted JSON, approved content only |
| `GET /api/proposals?status=&q=` | List, with counts per status |
| `POST /api/proposals` | Create from proposal JSON |
| `POST /api/proposals/generate` | Claude draft, saved as a new proposal |
| `GET /api/proposals/{id}` | Proposal, approval state, permissions |
| `PUT /api/proposals/{id}` | Save edits (`base_version` required; 409 on conflict) |
| `GET /api/proposals/{id}/gate` | Final Gate on the saved version (WARN keys for acknowledgement) |
| `POST /api/proposals/{id}/preview` | DRAFT PDF of the saved version |
| `POST /api/proposals/{id}/submit` | Draft → In review |
| `POST /api/proposals/{id}/return` | Reviewer: back to Draft |
| `POST /api/proposals/{id}/approve` | Reviewer approval |
| `GET /api/proposals/{id}/pdf` | Final PDF (approved version only) |
| `GET /api/proposals/{id}/versions[/{n}]` | Version list or snapshot |
| `GET /api/proposals/{id}/approvals` | Approval history |
| `GET /api/proposals/{id}/audit` | Audit log |

## Tests

```bash
cd backend && .venv/bin/python -m pytest -q    # 48 tests, isolated temp DB, no API calls
cd frontend && npm test                        # 20 tests
cd frontend && npm run build                   # production build
```

## AI drafting

- **Model:** `claude-opus-5-5`, at effort `high` for drafts and `low` for intake analysis.
  Server-side refusal fallback is on.
- **System prompt:** the skill's `SKILL.md` plus its guidelines, loaded verbatim from
  `backend/guidelines/` with a short web-app preamble (`app/prompts.py`). It is cached
  between calls.
- **Output checks:** intake analysis uses constrained decoding (`IntakeAnalysis` schema). The
  full `DraftOutput` schema is too large for the API's constrained-decoding grammar limit,
  so the draft call puts the JSON Schema in the prompt and the server validates the reply
  with Pydantic. If it fails, the exact errors go back to Claude for one correction turn;
  a second failure returns 502 and nothing is saved. The server then sets the entity,
  contacts, Deal ID, quote number, version and date from the intake form, rejects figures
  that were not uploaded, and runs the Final Gate.

## Layout

```
backend/
  engine/      skill engine, copied unchanged (render.py, gate.py, roster.py)
  assets/      fonts the engine uses
  guidelines/  skill SKILL.md and section guidelines (drafting system prompt)
  app/         main, auth, proposals, models, db, config, schema, builder, drafting, prompts, storage
  migrations/  Alembic
  tests/       API, drafting and workflow tests
frontend/
  src/pages/       Login, Dashboard, NewProposal, ProposalReview
  src/editor/      block editors, cover, review panel, approval dialog, PDF modal, history
  src/lib/         proposal JSON helpers, intake mapping
  src/integrations dealSource (manual now; Zoho in Phase 4)
```

When the skill is updated, copy its `engine/`, `assets/` and `guidelines/` over these
folders and re-run the tests.

## Phase 4: remaining Zoho work

- **Sign-in:** add Zoho OAuth 2.0 against `accounts.zoho.in`, with a server-side
  authorization code flow and the callback in `app/auth.py`. Restrict sign-in to
  @ingsol.com accounts and map roles (who is a reviewer) from a Zoho group or an admin
  list. Set `AUTH_MODE=zoho` and a real `SESSION_SECRET`, and remove the dev users from any
  deployed configuration.
- **CRM lookup:** add backend endpoints that search Deals and read Quotes through the Zoho
  CRM API (`www.zohoapis.in`). Tokens stay server-side. Confirm the API names of the Deal ID
  and quote number fields in Zoho Setup.
- **Frontend:** register a `zohoDealSource` in `frontend/src/integrations/dealSource.js` so the
  wizard can search deals and pre-fill the client, Deal ID, title and quote number.
- **Other entities:** add the INGSOL FZ-LLC and INGSOL LTD addresses (`app/entities.py`)
  once they are confirmed.
# Ingsol-proposal_writer
