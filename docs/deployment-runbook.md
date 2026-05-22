# Deployment Runbook — AI Recruitment System

## Document Control
- Version: 1.0
- Last updated: 2026-04-30
- Environment: Docker Compose (single-node)

---

## 1. Pre-deployment Checklist

| Check | Command / Action |
|---|---|
| Docker + Compose installed | `docker --version && docker compose version` |
| `.env` file created from example | `cp .env.example .env` then fill all values |
| `SECRET_KEY` is a strong random string | `python -c "import secrets; print(secrets.token_hex(32))"` |
| LLM endpoint reachable | `curl http://localhost:11434/api/tags` (Ollama) or set `OPENAI_API_KEY` |
| Ports 8000, 8080, 5432, 6379 free | `netstat -an \| grep -E '8000\|8080\|5432\|6379'` |

---

## 2. Required Environment Variables

Create `src/ai-recruitment-system/.env` with the following (never commit this file):

```dotenv
# Required — generate with: python -c "import secrets; print(secrets.token_hex(32))"
SECRET_KEY=<your-64-char-random-hex>

# LLM backend — choose one:
#   Option A: Local Ollama
MODEL=ollama/llama3.1
API_BASE=http://host.docker.internal:11434
OPENAI_API_KEY=ollama

#   Option B: OpenAI cloud
# MODEL=gpt-4o-mini
# API_BASE=https://api.openai.com/v1
# OPENAI_API_KEY=sk-...

# Optional
APIFY_API_TOKEN=
DOCS_URL=          # leave empty to disable /docs in production
REDOC_URL=         # leave empty to disable /redoc in production
ALLOWED_ORIGINS=https://yourdomain.com
```

---

## 3. First-Time Deployment

```bash
cd src/ai-recruitment-system

# 1. Build all images
docker compose -f docker/docker-compose.yml build --no-cache

# 2. Start infrastructure first
docker compose -f docker/docker-compose.yml up -d redis postgres

# 3. Wait for postgres to be ready (~5s), then start api + worker
docker compose -f docker/docker-compose.yml up -d api worker

# 4. Start frontend (waits for api healthcheck)
docker compose -f docker/docker-compose.yml up -d frontend

# 5. Verify all services healthy
docker compose -f docker/docker-compose.yml ps
```

**Expected output (all services healthy):**
```
NAME        STATUS              PORTS
api         Up (healthy)        0.0.0.0:8000->8000/tcp
frontend    Up                  0.0.0.0:8080->80/tcp
postgres    Up                  0.0.0.0:5432->5432/tcp
redis       Up (healthy)        0.0.0.0:6379->6379/tcp
worker      Up (healthy)        —
```

---

## 4. Smoke Tests After Deployment

```bash
# Health endpoint (no auth required)
curl http://localhost:8000/health
# Expected: {"status":"ok"}

# List available crews (no auth required)
curl http://localhost:8000/api/v1/crews/
# Expected: {"crews":[...]} with recruitment_screening, scoring_only, marketing_strategy

# Get a JWT token (replace SECRET_KEY in test only)
TOKEN=$(python -c "
import os; os.environ['SECRET_KEY']='<your-secret-key>'
import sys; sys.path.insert(0,'src/ai-recruitment-system')
from app.core.security import create_access_token
print(create_access_token({'sub':'ops-test'}))
")

# Submit a job (auth required)
curl -X POST http://localhost:8000/api/v1/jobs/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"crew_key":"recruitment_screening","input":{"cv_text":"Alice, 5yr Python","job_description":"Backend Engineer"}}'
# Expected: {"job_id":"<uuid>","crew_key":"recruitment_screening"}
```

---

## 5. Updating / Re-deploying

```bash
# Pull latest code
git pull origin main

# Rebuild changed services only
docker compose -f docker/docker-compose.yml build api worker

# Rolling restart (zero-downtime is NOT guaranteed in single-node compose)
docker compose -f docker/docker-compose.yml up -d --no-deps api worker
```

---

## 6. Rollback Procedure

```bash
# Tag before deploying
git tag pre-deploy-$(date +%Y%m%d)

# To rollback to previous tag
git checkout <previous-tag>
docker compose -f docker/docker-compose.yml build api worker
docker compose -f docker/docker-compose.yml up -d --no-deps api worker
```

---

## 7. Logs & Monitoring

```bash
# Tail all service logs
docker compose -f docker/docker-compose.yml logs -f

# Tail specific service
docker compose -f docker/docker-compose.yml logs -f api

# Check celery worker tasks
docker compose -f docker/docker-compose.yml exec worker \
  celery -A app.infrastructure.queue.celery_app inspect active
```

Log format is structured JSON (see `app/core/logging.py`). Fields: `timestamp`, `level`, `logger`, `agent`, `trace_id`, `status`, `latency`, `message`.

---

## 8. Backup & Recovery

```bash
# Backup Postgres data
docker compose -f docker/docker-compose.yml exec postgres \
  pg_dump -U user recruitment > backup_$(date +%Y%m%d).sql

# Restore
cat backup_20260430.sql | docker compose -f docker/docker-compose.yml exec -T postgres \
  psql -U user recruitment
```

---

## 9. Shutdown

```bash
# Stop all services (preserves volumes)
docker compose -f docker/docker-compose.yml down

# Stop and delete volumes (DESTRUCTIVE — wipes database)
docker compose -f docker/docker-compose.yml down -v
```
