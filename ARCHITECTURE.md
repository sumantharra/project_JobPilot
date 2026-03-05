# JobPilot Architecture

## 1. Overview

JobPilot is an AI-powered job application co-pilot that:

- Aggregates jobs from multiple sources into a **unified job feed**
- Scores jobs against a user’s **resume profile**
- Generates **tailored resumes** and **cover letters**
- Builds an **Apply Pack** (resume + cover letter + Q&A + apply link)
- Provides an **application tracker** with statuses, notes, and follow-ups
- Enforces **assisted apply only** and **no fabrication of experience**

---

## 2. High-Level Components

### 2.1 Frontend (React + TypeScript)

- **Tech**: React + TypeScript (SPA or Next.js)
- **Responsibilities**:
  - User authentication UI
  - Source connection management (RSS, CSV, Gmail, career sites)
  - Unified job feed (filters, search, sort by match)
  - Resume upload and management
  - Tailored resume and cover letter editors
  - Apply Pack view and “Open application” button
  - Application tracker (board/list with statuses, notes, follow-ups)

Typical structure:

- `frontend/src/pages` – routes (jobs, applications, settings, auth)
- `frontend/src/components` – shared UI components
- `frontend/src/api` – API client wrappers

---

### 2.2 Backend API (FastAPI)

- **Tech**: FastAPI, Python 3.11
- **Responsibilities**:
  - JWT authentication & user management
  - REST endpoints for:
    - Sources & connectors
    - Jobs (list, filters, details, apply URL)
    - Resumes (upload, parse, list, versions)
    - Match scoring & explanations
    - Tailored resumes & cover letters
    - Apply Packs
    - Applications & tracker
  - Orchestrating **background jobs** via Redis + worker
  - Enforcing business rules:
    - Assisted apply only (no auto-submit bots)
    - Human review before submission
    - No fabrication of resume experience

Suggested layout:

- `backend/app/main.py` – FastAPI app entrypoint
- `backend/app/core` – config, security, settings
- `backend/app/api` – route definitions
- `backend/app/models` – ORM / Pydantic models
- `backend/app/services` – business logic
- `backend/app/ai` – AI orchestration layer

---

### 2.3 Connectors (Job Ingestion Engine)

- **Pattern**: Plugin-based architecture.
- Each connector implements:

  ```python
  search_jobs()
  get_job_details()
  get_apply_link()
  ```

- **Types of sources**:
  - RSS feeds
  - Career sites (Greenhouse, Lever, Workday-style)
  - CSV uploads
  - (Later) Gmail job alerts and other portals (if ToS-compliant)

- **Location**:
  - `connectors/base.py` – `JobConnector` abstract base class
  - `connectors/<source_name>.py` – concrete implementations

Connectors are used from backend/services and background workers, never called directly by the frontend.

---

### 2.4 Background Workers

- **Tech**: Python worker process (Celery/RQ/custom) using Redis as a broker.
- **Responsibilities**:
  - Periodic job ingestion from all connectors
  - Normalization & deduplication pipeline for jobs
  - Heavy AI tasks (resume parsing, bulk scoring, tailoring)
  - Follow-up reminders (for applications)

- **Location**:
  - `workers/worker.py` – worker entrypoint (later integrated with a real task queue)

---

### 2.5 AI Service Layer

- **Tech**: OpenAI / ChatGPT-style APIs
- **Responsibilities**:
  1. Extract job requirements from job descriptions
  2. Extract skills from resumes and job descriptions
  3. Compute match scores between resume and job
  4. Provide “Why Matched” explanations and skill gaps
  5. Rewrite resume bullets (tailoring) without adding new experience
  6. Generate job-specific cover letters with configurable tone
  7. Generate Q&A responses for Apply Packs

- **Safety & Policies**:
  - Only rewrite existing resume facts
  - No creation of new employers/roles/skills the user did not provide
  - Outputs must always be reviewed by the user before use

- **Location**:
  - `backend/app/ai/` – clients, prompts, orchestration

---

### 2.6 Storage Layer

**Primary database**: PostgreSQL

Core tables:

- `users`  
  - `id`, `email`, `name`, `preferences`

- `sources`  
  - `id`, `user_id`, `type`, `config`, `last_fetched_at`

- `jobs`  
  - `id`, `source`, `title`, `company`, `location`, `description`, `apply_url`, `fingerprint_hash`

- `resumes`  
  - `id`, `user_id`, `structured_data`

- `resume_versions`  
  - `id`, `resume_id`, `job_id`, `tailored_content`, `created_at`

- `applications`  
  - `id`, `job_id`, `resume_version_id`, `status`, `notes`, `follow_up_date`

**Redis**:

- Task queue backend for workers
- Optional caching for AI results and match scores

---

## 3. Key Flows

### 3.1 Job Ingestion & Unified Feed

1. Worker triggers `search_jobs()` on each connector.
2. Raw jobs are normalized into the `jobs` table.
3. Deduplication runs using `fingerprint_hash` (e.g. company + title + URL).
4. Backend exposes `GET /jobs` with filters, search, and pagination.
5. Frontend displays the unified job feed with filters and search.

---

### 3.2 Resume Profile & Match Scoring

1. User uploads resume via frontend.
2. Backend stores file and parses it into `resumes.structured_data`.
3. For a given job + resume:
   - AI extracts skills from both.
   - AI computes match score and generates:
     - “Why this matches you”
     - “Skill gaps” list.
4. Backend exposes APIs to get scores/explanations.
5. Frontend shows scores on the job list and explanations on job details.

---

### 3.3 Tailored Resume & Cover Letter

1. User clicks “Tailor resume” on a job.
2. AI extracts requirements from the job description.
3. AI maps existing resume bullets to those requirements.
4. AI rewrites bullets (better phrasing, order) **without adding new experience**.
5. Result is stored as a `resume_versions` entry for that job.
6. AI generates a draft cover letter using the resume + JD (+ tone).
7. Frontend shows editors for tailored resume + cover letter, with version history.

---

### 3.4 Apply Pack & Application Tracker

1. Backend builds an Apply Pack:
   - Tailored resume version
   - Cover letter
   - Q&A responses
   - External `apply_url`
2. Frontend shows the Apply Pack on job detail with a big “Open application page” button.
3. When the user applies, they update an `application` record with status, notes, and follow-up date.
4. Tracker UI reads `applications` and visualizes them by status (Saved, Applied, Interview, Offer, Rejected).

---

## 4. Non-Functional Principles

- **Compliance**: No automated form submissions; only assisted apply.
- **Integrity**: No fabrication of resume experience; AI is constrained to user-provided facts.
- **Scalability**: Connectors are modular, workers handle heavy tasks, DB schema is normalized.
- **User Control**: All AI outputs are editable and must be reviewed by the user.

---

## 5. Project Folder Structure (Target)

```text
project_JobPilot/
  backend/
    app/
      main.py
      core/
        config.py
      api/
      models/
      services/
      ai/
    requirements.txt

  frontend/
    src/
      pages/
      components/
      api/

  connectors/
    base.py
    rss.py
    greenhouse.py
    ...

  workers/
    worker.py

  infra/
    docker-compose.yml
    Dockerfile.backend

  README.md
  ARCHITECTURE.md
```

