# Autonomous Product Review

Industrial quality review coordination for **PS 190**: process inspection images and
supplied defect annotations, create evidence-linked findings and a draft review
queue, and require inspector approval before finalizing outcomes.

## Features

- Three-panel inspection station: four-image inspection, manager inboxes, and report.
- Four simultaneous image analyses, with automatic progression to the next group.
- Continuous dataset processing until paused or stopped.
- Original-image evidence, supplied defect masks, suggested regions, and inspector corrections.
- Product-presence checks before defect assessment.
- Issue-based routing into contamination, cracks, surface, components, and general review inboxes.
- A visible thumbnail handoff tied to actual persisted manager assignments.
- Batch flags, precautionary holds, maintenance investigations, and follow-up records.
- Inspector decisions, rechecks, batch review sign-off, and downloadable reports.
- Dashboard metrics, activity charts, review outcomes, manager workload, and searchable batch reports.

## Stack

| Layer | Technology |
| --- | --- |
| Frontend | Next.js 14, React, TypeScript, Tailwind CSS |
| Backend | Python 3.11+, FastAPI, Pydantic, SQLAlchemy |
| Storage | PostgreSQL for records; local files for inspection images and masks |
| Vision model | OpenAI GPT-4o mini, configurable through `OPENAI_MODEL_VISION` |
| Authentication | NextAuth with backend bearer authentication |
| Local services | PostgreSQL and Redis through Docker or native local services |

The quality workflow uses asynchronous backend tasks and explicit routing rules.
Its separate panels do not represent separate autonomous LLM agents.

## Run locally

Install Node.js 20+, Python 3.11+, and Docker with Compose. The startup script also
supports native PostgreSQL and Redis when their command-line tools are available.

```bash
./scripts/start.sh
```

The first run creates local environment files and installs dependencies. Configure
`OPENAI_API_KEY` and `OPENAI_MODEL_VISION=gpt-4o-mini` in `.env`, then restart to enable
vision calls. Do not commit your API keys or local environment files.

| Page | Address |
| --- | --- |
| Inspection station | http://localhost:3000/quality |
| Batch dashboard | http://localhost:3000/quality/dashboard |
| Backend API documentation | http://localhost:8000/docs |

The development database seed includes the local demo account `demo@argus.app` with password
`argusdemo123`. These are development-only credentials. Use individual accounts and
appropriate secrets for any deployment. See [the runbook](docs/runbook.md) for setup details.

## Dataset and inspection modes

Import product images through the interface. The local dataset importer recognizes
MVTec-style `~/Downloads/transistor/test`, `~/Downloads/tile/test`, and
`~/Downloads/wood/test` folders, together with matching `ground_truth` masks.
Dataset files are not bundled with this repository; obtain them separately and
check their applicable license before use.

- **Annotation-assisted:** supplied labels and masks are explicit evidence inputs.
  Model commentary can explain or disagree with them. These are not independent AI detections.
- **AI-only:** the vision model examines target images with product criteria and
  available known-good references; its findings and boxes remain unverified suggestions.
- **Manual:** an inspector records decisions without model analysis.

Unknown or missing product presence blocks automatic acceptance. Inspectors can
correct regions, accept, reject, request rework, or request reinspection. Reports
remain drafts until their review requirements are satisfied.

Repeated photographs can reuse a stored analysis after a short preparation phase.
Each occurrence retains its source asset ID, gets its own inspection record, and
requires its own inspector decision. Repeated processing is not independent model
validation or evidence of additional physical products. Recheck bypasses reuse.

## Workflow and API

All quality endpoints are owner-scoped and require authentication.

| Endpoint | Purpose |
| --- | --- |
| `GET /quality/workspace` | Current runs, inspections, evidence, events, and reports |
| `POST /quality/assets` | Import inspection images |
| `POST /quality/import-local` | Import supported local datasets and supplied annotations |
| `POST /quality/runs` | Start a finite or continuous inspection run |
| `POST /quality/runs/{id}/control` | Pause, resume, or stop processing |
| `POST /quality/review-selection` | Record inspector decisions for up to four images |
| `POST /quality/inspections/{id}/recheck` | Request a fresh assessment |
| `GET /quality/manager-inboxes` | Read issue-group assignments |
| `POST /quality/runs/{id}/route` | Repair missing assignments idempotently |
| `POST /quality/batch-reports/{id}/decision` | Finalize batch report review |

Manager inboxes are internal role-based folders. The platform does not send external
email or control physical factory equipment. Backend restarts pause active runs and
mark interrupted analyses for rechecking. Run one backend worker for this prototype.

## Validation

```bash
cd backend
.venv/bin/python -m pip install -e '.[dev]' aiosqlite
.venv/bin/python -m pytest tests/test_quality.py -q
```

```bash
cd frontend
npm run build
```

The API tests use isolated SQLite storage and controlled model responses; they do
not measure live-model detection accuracy. Coverage includes concurrent processing,
source provenance, approval guards, routing repair, cache behavior, and continuous
processing beyond 150 inspection occurrences.

## Project structure

- `frontend/components/quality/`: inspection, inboxes, dashboard, and reports.
- `frontend/app/quality/`: quality routes and styles.
- `backend/app/api/quality.py`: authenticated quality API.
- `backend/app/services/quality_service.py`: analysis and inspection execution.
- `backend/app/services/quality_routing.py`: issue-group assignment rules.
- `backend/app/models/quality.py`: persisted quality records.
- `backend/tests/test_quality.py`: quality workflow integration tests.
- `database/`, `scripts/`: database setup and local startup.

The codebase retains earlier browser-agent modules under `agents/`, `integrations/`,
and legacy frontend routes. The dedicated product starts at the quality inspection station.

## Prototype limitations

The general-purpose vision model can miss defects or produce inaccurate regions.
No production-grade accuracy claim is made. Supplied-mask results must be kept
separate from independent predictions during evaluation. Factory deployment needs
product-specific validation, access controls, operational integration, and measured
false-negative performance.
