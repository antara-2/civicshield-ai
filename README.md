# CivicShield AI — IMPACTX 2026 Challenge 10

**An advanced full-stack municipal incident-management prototype** built around a 140-report synthetic heavy-weather scenario. It emphasizes explainable prioritization, auditable clustering, cross-department ownership, privacy-first media processing, abuse resistance, offline field synchronization and honest public progress.

> **Important:** This is a local demonstration, not a connection to a real municipal corporation. All reports, coordinates, ward boundaries, weather and department states are synthetic. The demo bearer token is not production authentication. Do not use it to dispatch real crews.

## Quick start

### Option 1 — Local Python / SQLite

Requires Python 3.11+ and Tesseract OCR for text-in-image detection. The app remains usable if OCR is unavailable, but redaction coverage is lower.

```bash
cd civicshield_ai
python -m venv .venv
# Windows PowerShell: .venv\Scripts\Activate.ps1
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
# Copy .env.example to .env and adjust values if desired.
uvicorn app.main:app --reload
```

Open `http://127.0.0.1:8000`. The database is SQLite at `data/civicshield.db`, and only sanitized media artifacts are stored under `data/media/`. First startup seeds 140 synthetic reports; subsequent starts preserve data. To reset, stop the server and delete `data/civicshield.db` (and any sanitized media in `data/media/`) before restarting.

**Demo access token:** `demo-civicshield-token` (also configured in the UI settings dialog). This token is intentionally visible in the local prototype and is not suitable for production.

### Option 2 — Docker + PostgreSQL/PostGIS

```bash
cd civicshield_ai
docker compose up --build
```

Open `http://localhost:8000`. Docker Compose uses PostGIS for the database service and mounts a persistent media volume. The app uses latitude/longitude columns and a transparent haversine rule in this prototype; a production extension can add native PostGIS geometries and spatial indexes.


## Public demo deployment (Render)

This repository includes a `render.yaml` blueprint and a Dockerfile configured to bind to Render's `PORT`. To create a public demo URL:

1. Extract the project ZIP on your computer.
2. Create a GitHub repository and upload the **contents** of the `civicshield_ai` folder (including `render.yaml`, `Dockerfile`, `app/`, `data/`, `docs/`, and `requirements.txt`). Do not upload `.venv/`, a local database, real reports, or private images.
3. In Render, connect GitHub, choose **New → Blueprint**, and select the repository containing `render.yaml`. Review the configuration and deploy. If the Blueprint option is unavailable, create a **Web Service** from the repository, choose **Docker**, and use `./Dockerfile`.
4. Wait until the deploy is marked live. Open the `onrender.com` URL shown in the Render dashboard and test `/api/health` and the main page. Put that actual URL in the PowerPoint.

**Free-demo limitations:** the free web service may sleep after inactivity and its local filesystem is ephemeral. The SQLite database, uploaded demo media, and changes may be reset after a restart or redeploy; the app re-seeds its synthetic demo dataset on startup. Do not use real citizen data. The hard-coded demo token is public and is not production authentication.

## Main application surfaces

- **Command center:** KPIs, schematic incident map, risk-ranked queue, SLA watch and fault lab.
- **Incident queue:** report cluster evidence, transparent risk components, source freshness, separate lifecycle states, per-department SLA clocks, human overrides, merge/split and handoff decisions.
- **Citizen report form:** anonymous low-bandwidth reporting, optional image, redaction pipeline and a withdrawal token for media consent.
- **Field operations:** tasks, offline-friendly PWA shell, IndexedDB local queue for citizen submissions, geotagged field proof and revision-aware conflict handling.
- **Ward equity & SLA:** ward comparison with sample-size warnings and an explicit limitation note.
- **Audit & evaluation:** SHA-256 hash chain verification, 22 live checks, audit-event explorer and project data/threat docs.
- **Public status:** sanitized generalized progress without exact coordinates, raw user text, reporter identifiers or evidence URLs.

## Architecture

```text
React-free single-page client (semantic HTML/CSS/JS; no external runtime bundles)
    ├── Accessible staff dashboard + public report/status forms
    ├── PWA shell cache + IndexedDB offline report queue
    └── FastAPI JSON API
          ├── SQLAlchemy data layer → SQLite (local) or PostgreSQL (Compose)
          ├── Space/time/lexical cluster engine with per-link explanations
          ├── Explainable weighted risk formula and sensitivity endpoint
          ├── Multi-department routing, individual SLA rows and escalation watchdog
          ├── OpenCV face boxes + Tesseract OCR heuristics + EXIF-free re-encoding
          ├── Revision-aware field-task sync and conflict record
          └── SHA-256 previous-hash chained audit events
```

The frontend is a lightweight single-page web application rather than a compiled React bundle to keep installation small and the offline path straightforward. API routes are documented at `http://localhost:8000/docs`. The app uses no live weather, map tile, geocoding, email, SMS, or municipal integration.

## Risk formula

The default settings are illustrative and normalize to 100%:

`R = 0.30H + 0.20S + 0.15P + 0.15W + 0.10T + 0.10C`

- `H`: hazard severity (category-based)
- `S`: proximity to a sensitive site
- `P`: estimated exposed population
- `W`: weather conditions, discounted and labelled stale when source age exceeds six hours
- `T`: hours outstanding, capped at 100
- `C`: recurrence count, capped at 100

Every risk output returns the factor values, normalized weights, explanatory text, confidence/uncertainty, source and freshness. Scores are not safety certifications; emergency overrides take precedence. The sensitivity endpoint is non-mutating.

## API map

Public endpoints:

- `GET /api/health`
- `GET /api/public/incidents`
- `POST /api/reports`
- `POST /api/reports/{ref}/media` (redact first; store only processed output)
- `POST /api/reports/{ref}/withdraw-consent`
- `POST /api/media/redact` (preview-only, not persisted)

Official demo routes require `Authorization: Bearer demo-civicshield-token`:

- `GET /api/dashboard`, `GET /api/incidents`, `GET /api/incidents/{id}`
- `POST /api/recluster` (dry run; no report or notification mutation)
- `POST /api/incidents/{id}/recalculate-risk`, `/risk-sensitivity`, `/assign`, `/departments/{department}/decision`, `/split`, `/override`
- `POST /api/incidents/merge`
- `POST /api/system/escalations/sweep`
- `GET /api/field/tasks`, `POST /api/field/tasks`, `POST /api/field/sync`
- `GET /api/equity`, `GET /api/audit`, `GET /api/audit/verify`
- `GET /api/evaluation/cases`, `POST /api/evaluation/run`, `POST /api/demo/fault`

## Tests

Install requirements, then run:

```bash
pytest -q
```

The test suite creates an isolated in-memory SQLite database and seeds the demonstration dataset. The web dashboard can also run its live **22-case evaluation harness** against whichever database is active. Not every check is a full end-to-end or security check; its output labels the distinction.

## Privacy pipeline

`app/privacy.py` processes images in memory, performs frontal-face detection, OCRs text lines where Tesseract is installed, blurs likely phone/plate lines, and re-encodes a fresh JPEG with no EXIF. The original input is never written to disk by the upload endpoint. The endpoint returns the detection counts and an explicit warning because heuristic detectors can miss sensitive content. Generalizable redaction recall requires a labelled and representative image set; this prototype does not claim perfect recall.

## Operational / production gaps

Before any real deployment: implement institutional authentication and authorization, CSRF/session protections where applicable, distributed rate limiting, database migration management, immutable external audit checkpoints, privacy/retention governance, secure media storage and deletion, a real ward-boundary source, validated department ownership rules, signed weather sources, labelled multilingual redaction evaluation, assistive-technology testing and incident-response playbooks. See `docs/THREAT_MODEL.md`, `docs/STATE_MACHINES.md`, and `docs/EVALUATION.md`.
