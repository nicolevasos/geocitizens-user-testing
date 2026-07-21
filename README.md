# GeoCitizens — Single FastAPI + PostgreSQL

This is the complete GeoCitizens study application in one deployable service. FastAPI serves the five-tab frontend, IGAC cadastral API integration, exact Shapely IoU, WHISP workflow, Stage 1 repair alternatives, Stage 3 active learning, study mode, developer map, and session analytics. All shared research data is stored in PostgreSQL when `DATABASE_URL` is set.

## Application URLs

- `/` — GeoCitizens app
- `/study` — participant-link generator
- `/analytics` — session analytics dashboard
- `/debug` — cadastral cache and geometry dashboard
- `/docs` — FastAPI API documentation
- `/health` — application and database health

## Database behavior

- **Render/production:** `DATABASE_URL` points to managed Render PostgreSQL.
- **Local PostgreSQL:** use `docker compose up --build`.
- **Local fallback:** if `DATABASE_URL` is absent, GeoCitizens uses `data/cadastre_cache.sqlite3` so the app remains easy to run without Docker.

PostgreSQL stores:

- cadastral cache and matched IGAC geometries
- active-learning review queue and farmer labels
- evidence-layer weights
- repair submissions, choices, and repair-strategy statistics
- participant sessions and event logs
- study questionnaires

## Run locally with PostgreSQL — recommended

Install Docker Desktop, then run from the project root:

```bash
docker compose up --build
```

Open:

```text
http://127.0.0.1:5050/
```

PostgreSQL is persisted in the Docker volume `geocitizens_postgres`.

Stop services:

```bash
docker compose down
```

Stop and delete the local database volume:

```bash
docker compose down -v
```

## Run locally without Docker

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r backend/requirements.txt
python3 -m uvicorn backend.app:app --reload --host 127.0.0.1 --port 5050
```

With no `DATABASE_URL`, this uses the SQLite fallback.

## Run locally against an existing PostgreSQL server

```bash
export DATABASE_URL='postgresql://geocitizens:password@127.0.0.1:5432/geocitizens'
python3 -m uvicorn backend.app:app --reload --host 127.0.0.1 --port 5050
```

The schema and default scoring weights are initialized automatically at startup.

## Deploy to Render

### 1. Push the project to GitHub

The repository root must contain `render.yaml` and `Dockerfile`.

### 2. Create a Render Blueprint

In Render:

1. Select **New → Blueprint**.
2. Connect the GitHub repository.
3. Render reads `render.yaml` and creates:
   - one Docker web service
   - one managed PostgreSQL database
4. Approve the resources and deploy.

The Blueprint injects the database's private connection string into `DATABASE_URL`. No database password is committed to GitHub.

### 3. Verify deployment

Open:

```text
https://<your-service>.onrender.com/health
```

Expected fields include:

```json
{
  "status": "ok",
  "database_status": "connected",
  "database": "PostgreSQL (DATABASE_URL)"
}
```

Then use:

```text
https://<your-service>.onrender.com/study
```

to generate participant links.

## Render configuration

`render.yaml` provisions a `basic-256mb` managed PostgreSQL instance and passes its internal connection string to the web service through:

```yaml
- key: DATABASE_URL
  fromDatabase:
    name: geocitizens-postgres
    property: connectionString
```

The FastAPI container binds to Render's `$PORT` automatically.

## Migrating existing SQLite study data

Set a PostgreSQL URL and run:

```bash
export DATABASE_URL='postgresql://...'
python3 -m backend.migrate_sqlite_to_postgres \
  --sqlite data/cadastre_cache.sqlite3
```

To replace all destination records first:

```bash
python3 -m backend.migrate_sqlite_to_postgres \
  --sqlite data/cadastre_cache.sqlite3 \
  --truncate
```

Create a backup before migration.

## Participant links

Example:

```text
https://<your-service>.onrender.com/?study=1&participant=P001&condition=multi_candidate
```

Use anonymous study codes rather than names or email addresses.

## Research data protection

Before external testing:

- configure an appropriate managed database plan and backups
- restrict researcher dashboards if the deployment is public
- use anonymous participant codes
- document retention and deletion periods in the consent material
- never commit credentials or `.env` files

## Tests

```bash
python3 -m pip install pytest
pytest -q
```

## Geometry revalidation after manual edits

Manual vertex edits are treated as a new geometry. When editing is saved, the application now:

1. Clears the previous Stage 2 indicators.
2. Re-runs Stage 1 topology checks.
3. Blocks confirmation and opens the repair workflow when the edited polygon is invalid.
4. Recalculates A2, A3, A4, confidence, and the confidence interval with a forced cache refresh when the geometry is valid.
5. Recalculates the indicators again after an automatic repair is accepted.

The plain-language card remains visible by default. The score and geometry cards are available through **Ver indicadores técnicos** without being moved or reordered in the DOM.
