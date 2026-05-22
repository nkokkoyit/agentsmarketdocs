# Threat Model — AI Recruitment System

## Document Control
- Version: 1.0
- Last updated: 2026-04-30
- Methodology: STRIDE
- Scope: API + Worker + LLM pipeline

---

## 1. System Overview

```
Internet → [nginx] → [FastAPI API] → [Redis Queue] → [Celery Worker] → [LLM (CrewAI)]
                           │
                      [PostgreSQL]
```

**Trust boundaries:**
- **External** (internet): Unauthenticated clients
- **DMZ**: nginx reverse proxy, FastAPI (JWT-authenticated)
- **Internal**: Redis, Postgres, Worker, LLM (not internet-exposed)

---

## 2. Assets

| Asset | Sensitivity | Location |
|---|---|---|
| JWT Secret Key | Critical | `SECRET_KEY` env var |
| Apify API Token | High | `APIFY_API_TOKEN` env var |
| OpenAI API Key | High | `OPENAI_API_KEY` env var |
| PostgreSQL credentials | High | `DATABASE_URL` env var |
| Candidate CV data | High | In-memory + PostgreSQL |
| LLM outputs (scores, decisions) | Medium | PostgreSQL / Redis |
| Workflow YAML templates | Low | `workflows/` directory |

---

## 3. STRIDE Threat Matrix

### 3.1 Spoofing

| ID | Threat | Component | Mitigation | Status |
|---|---|---|---|---|
| S-01 | Forged JWT token | API endpoints | HMAC-HS256 signature verification via `python-jose`. `verify_token()` applied to all mutating endpoints. | ✅ Mitigated |
| S-02 | Token replay after expiry | API endpoints | 30-minute expiry enforced via `exp` claim in `create_access_token()` | ✅ Mitigated |
| S-03 | Worker impersonation on Redis queue | Celery/Redis | Redis not exposed externally (internal Docker network only). No authentication on Redis — acceptable for internal-only deployment. | ⚠️ Residual (acceptable) |

### 3.2 Tampering

| ID | Threat | Component | Mitigation | Status |
|---|---|---|---|---|
| T-01 | Tampered job payload via API | POST /api/v1/jobs/ | Pydantic schema validation (JobSubmitRequest). Extra fields ignored. | ✅ Mitigated |
| T-02 | Modified YAML workflow templates | `workflows/` | Templates loaded at startup from local filesystem; no runtime upload API exposed. | ✅ Mitigated |
| T-03 | SQL injection via database_url | PostgreSQL | SQLAlchemy ORM used; parameterized queries. No raw SQL with user input. | ✅ Mitigated |
| T-04 | Task ID forgery for result access | GET /api/v1/results/{task_id} | JWT required; UUID format. 404 returned for unknown IDs. | ✅ Mitigated |

### 3.3 Repudiation

| ID | Threat | Component | Mitigation | Status |
|---|---|---|---|---|
| R-01 | No audit trail for job submissions | API | Structured JSON logging with `trace_id = job_id` on all job/task events via `app/core/logging.py` | ✅ Mitigated |
| R-02 | No immutable log store | Logging | Logs to stdout only; no tamper-proof storage. | ⚠️ Open — recommend shipping logs to external SIEM |

### 3.4 Information Disclosure

| ID | Threat | Component | Mitigation | Status |
|---|---|---|---|---|
| I-01 | Stack trace leakage in error response | FastAPI | `docs_url` and `redoc_url` disabled by default in production (empty env var). Exception handlers return only `detail` message. Test `test_404_does_not_leak_stack_trace` enforces this. | ✅ Mitigated |
| I-02 | Secret key in version control | config.py | `field_validator` rejects `changeme` / empty keys. `.env` in `.gitignore`. History scrubbed via `git-filter-repo`. | ✅ Mitigated |
| I-03 | Candidate data in LLM prompt logs | Worker / CrewAI | CrewAI verbose logging disabled by default. Sensitive CV text logged only at DEBUG level. | ⚠️ Residual — audit CrewAI internal logging |
| I-04 | CORS wildcard exposes API | CORS Middleware | `ALLOWED_ORIGINS` env var — defaults to `http://localhost:3000`. Production must set to exact domain. | ✅ Mitigated (config required) |

### 3.5 Denial of Service

| ID | Threat | Component | Mitigation | Status |
|---|---|---|---|---|
| D-01 | Brute-force / flood on POST endpoints | POST /jobs/, POST /tasks/ | `@limiter.limit("20/minute")` via slowapi. Global default 200/minute. | ✅ Mitigated |
| D-02 | Oversized payload causes memory exhaustion | FastAPI | No explicit body size limit set. | ⚠️ Open — recommend adding `Content-Length` check or nginx `client_max_body_size` |
| D-03 | LLM API exhaustion via job spam | CrewAI/Worker | Rate limiting on submission endpoints (D-01). Celery queue provides backpressure. | ✅ Mitigated (partial) |

### 3.6 Elevation of Privilege

| ID | Threat | Component | Mitigation | Status |
|---|---|---|---|---|
| E-01 | Unauthenticated access to data endpoints | All /api/v1/ except /crews/ | `Depends(verify_token)` on all write and sensitive read endpoints. | ✅ Mitigated |
| E-02 | Prompt injection via CV/JD input | LLM (CrewAI) | No system-level instructions mixed with user content. Agent goals are static YAML-defined strings. | ⚠️ Residual — recommend input sanitization before LLM prompt construction |
| E-03 | Container escape via base image vuln | Docker | Use `python:3.10-slim`. Regular base image updates needed. | ⚠️ Open — set up Dependabot or image scanning (Trivy) |

---

## 4. Open Risks Summary

| ID | Severity | Description | Recommendation |
|---|---|---|---|
| R-02 | Medium | Logs not shipped to tamper-proof store | Integrate with ELK stack or cloud logging |
| I-03 | Medium | Candidate PII may appear in CrewAI debug logs | Audit and disable CrewAI verbose mode in production |
| D-02 | Medium | No max body size limit on API | Add nginx `client_max_body_size 1m` |
| E-02 | Medium | LLM prompt injection possible via CV text | Sanitize/truncate user inputs before passing to LLM |
| E-03 | Low | Container base images may have known CVEs | Enable Dependabot or Trivy scanning in CI |
| S-03 | Low | Redis has no auth in Docker Compose | Add `requirepass` to Redis for multi-tenant deployments |

---

## 5. Security Controls Already Implemented

| Control | Implementation |
|---|---|
| Authentication | JWT HS256, 30-min expiry, `verify_token` dependency |
| Authorization | All write endpoints require valid token |
| Rate limiting | slowapi 20/min on POST, 200/min global |
| Input validation | Pydantic v2 strict schemas on all request bodies |
| Secret management | Env vars only, validator rejects weak/default values |
| CORS | Explicit origin allowlist via `ALLOWED_ORIGINS` |
| Structured audit logging | JSON logs with trace_id on all job events |
| API surface minimization | /docs and /redoc disabled by default in production |
| Git history | Secrets scrubbed from all commits via git-filter-repo |
