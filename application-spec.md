# Application Specification

## Document Control
- Project: AI Recruitment System
- Last updated (UTC): 2026-05-10 18:10:16 UTC
- Source of truth: Current code under src/ai-recruitment-system/app

## 1. Purpose
AI-powered recruitment workflow that evaluates candidates through API + async processing.

## 2. Scope
- In scope:
  - FastAPI task/result APIs
  - Async orchestration with Celery worker
  - CrewAI path and fallback scoring path
- Out of scope:
  - Long-term persistence redesign
  - Production-grade observability hardening

## 3. Functional Requirements
- Accept candidate analysis task requests.
- Execute async processing via worker task.
- Return task status and analysis result.
- Support fallback scoring when CrewAI path is unavailable.

## 4. Non-Functional Requirements
- Reliability: worker retry with exponential backoff
- Configurability: environment-driven runtime settings
- Traceability: structured logging with trace_id and status

## 5. Component Responsibilities
- api: see src/ai-recruitment-system/app/api
- application: see src/ai-recruitment-system/app/application
- core: see src/ai-recruitment-system/app/core
- domain: see src/ai-recruitment-system/app/domain
- infrastructure: see src/ai-recruitment-system/app/infrastructure
- schemas: see src/ai-recruitment-system/app/schemas
- workers: see src/ai-recruitment-system/app/workers

## 6. Runtime Behavior
- API receives a task/job request and delegates orchestration to JobService.
- JobService validates crew metadata, creates job_id, and enqueues the Celery worker task.
- Worker updates lifecycle state and executes crew logic.
- Output is stored and returned through result endpoint polling.

## 7. Data Lifecycle
- Input data: crew-scoped payload from POST /api/v1/tasks/ or POST /api/v1/jobs/.
- Processing data: task status transitions plus orchestration output.
- Output data: result object served by GET /api/v1/results/{task_id} or GET /api/v1/jobs/{job_id}.
- Persistence note: repository is in-memory in current implementation.

## 8. Risks And Constraints
- Task repository is currently in-memory and non-persistent.
- Runtime behavior depends on environment variables and queue mode.
- CrewAI path may vary by local model/provider readiness.

## 9. Change Summary
- Last commit: 75d8683 | 2026-05-11 01:01:14 +0700 | nkokkoyit | chore: ignore all test files
- Files changed in last commit:
- .gitignore
