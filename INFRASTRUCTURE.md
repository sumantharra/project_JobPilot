# JobPilot Infrastructure

## 1. Goals

The infrastructure layer for JobPilot should:

- Provide a **simple local dev environment** (backend + DB + Redis).
- Be easy to run with **one command** (`docker compose up`).
- Keep concerns separated: **backend**, **frontend**, **database**, **cache/workers**.
- Be ready to extend for **staging/production** later (e.g. cloud DB, managed Redis).

---

## 2. Components

### 2.1 Backend (FastAPI)

- **Code**: `backend/app/main.py`
- **Entrypoint**: `uvicorn backend.app.main:app`
- **Config**: `backend/app/core/config.py`
  - Reads from env vars (with `.env` override):
    - `DATABASE_URL` – PostgreSQL connection string
    - `REDIS_URL` – Redis connection string

### 2.2 Database (PostgreSQL)

- **Image**: `postgres:16`
- **Default credentials** (local/dev only):
  - `POSTGRES_USER=postgres`
  - `POSTGRES_PASSWORD=postgres`
  - `POSTGRES_DB=jobpilot`
- **Usage**:
  - Stores all persistent data: users, sources, jobs, resumes, applications, etc.

### 2.3 Cache & Queue (Redis)

- **Image**: `redis:7`
- **Usage**:
  - Task queue backend for background workers (job ingestion, AI tasks).
  - Optional caching for match scores or AI responses.

### 2.4 Workers

- **Code**: `workers/worker.py`
- **Responsibility**:
  - Background tasks (job ingestion, AI processing, reminders).
- **Current state**:
  - Simple placeholder loop; later replaced with Celery/RQ or a custom queue.

---

## 3. Directory Layout

```text
project_JobPilot/
  backend/
    app/
      main.py          # FastAPI app
      core/
        config.py      # Settings (DB, Redis, env)
    requirements.txt    # Backend Python deps

  infra/
    docker-compose.yml  # Local stack (db, redis, backend)
    Dockerfile.backend  # Backend container build

  connectors/
    base.py             # Connector interface (plugin-style)

  workers/
    worker.py           # Background worker entrypoint
```

---

## 4. Local Development (Without Docker)

### 4.1 Python virtual environment

```bash
cd /Users/sumanthreddyarra/Desktop/job_pilot/project_JobPilot

python3 -m venv .venv
source .venv/bin/activate

pip install -r backend/requirements.txt
```

### 4.2 Running the backend

```bash
# From project root (with .venv activated)
uvicorn backend.app.main:app --reload
```

- Backend will be available at: `http://127.0.0.1:8000`
- Health check: `GET /health`

### 4.3 Environment variables (local)

Create a `.env` file at project root (or inside `backend/`) if needed:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/jobpilot
REDIS_URL=redis://localhost:6379/0
```

These are read by `backend/app/core/config.py`.

---

## 5. Local Development (With Docker)

### 5.1 docker-compose

- **File**: `infra/docker-compose.yml`
- Starts:
  - `db` – Postgres
  - `redis` – Redis
  - `backend` – FastAPI app running with uvicorn

From the `infra` directory:

```bash
cd infra
docker compose up --build
```

Services:

- Backend: `http://127.0.0.1:8000`
- Postgres: `localhost:5432`
- Redis: `localhost:6379`

### 5.2 Backend Dockerfile

- **File**: `infra/Dockerfile.backend`
- Responsibilities:
  - Use `python:3.11-slim`
  - Copy project into `/app/project_JobPilot`
  - Install `backend/requirements.txt`
  - Run `uvicorn backend.app.main:app --host 0.0.0.0 --port 8000`

---

## 6. Environments

Currently, only **local/dev** is defined. Future environments:

- **Staging**:
  - Separate Postgres and Redis instances.
  - Docker images pushed to a registry.
  - Deployed via CI/CD.
- **Production**:
  - Managed Postgres (e.g. RDS, Cloud SQL).
  - Managed Redis (e.g. ElastiCache).
  - Container runtime (Kubernetes, ECS, etc.).
  - Secrets management (e.g. AWS Secrets Manager, Vault).

---

## 7. Next Infrastructure Steps

- Add **database migration tool** (e.g. Alembic) and migration commands.
- Add **worker container/service** to `docker-compose.yml`.
- Add **frontend** service to `docker-compose.yml` once the React app exists.
- Introduce **environment-specific configs** (`.env.local`, `.env.staging`, `.env.prod`).

