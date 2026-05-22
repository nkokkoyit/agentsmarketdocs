# API Specification

## Document Control
- Project: AI Recruitment System
- Last updated (UTC): 2026-05-10 18:10:16 UTC
- Base path: /api/v1

## 1. API Summary
Generated from FastAPI endpoint modules under app/api/v1/endpoints.

## 2. Endpoint Matrix
| Method | Path | Handler | Notes |
|---|---|---|---|
| POST | /api/v1/admin | create_provider | Defined in admin.py (status=201; response=LLMProviderResponse) |
| GET | /api/v1/admin | list_providers | Defined in admin.py |
| GET | /api/v1/admin/{name} | get_provider | Defined in admin.py (response=LLMProviderResponse) |
| PUT | /api/v1/admin/{name} | update_provider | Defined in admin.py (response=LLMProviderResponse) |
| DELETE | /api/v1/admin/{name} | delete_provider | Defined in admin.py (status=204) |
| GET | /api/v1/admin_llm_check/config | get_llm_config | Defined in admin_llm_check.py (response=LLMConfigResponse) |
| POST | /api/v1/auth/register | register | Defined in auth.py (status=201; response=RegisterResponse) |
| POST | /api/v1/auth/token | login | Defined in auth.py |
| GET | /api/v1/auth/me | me | Defined in auth.py (response=MeResponse) |
| GET | /api/v1/crews/ | list_crews | Defined in crews.py (response=CrewListResponse) |
| POST | /api/v1/jobs/ | submit_job | Defined in jobs.py (status=202; response=JobSubmitResponse) |
| GET | /api/v1/jobs/ | list_jobs | Defined in jobs.py (response=JobListResponse) |
| GET | /api/v1/jobs/{job_id} | get_job_status | Defined in jobs.py (response=JobStatusResponse) |
| GET | /api/v1/results/{task_id} | get_result | Defined in result.py (response=TaskResultResponse) |
| POST | /api/v1/tasks/ | create_task | Defined in task.py (status=202; response=TaskCreateResponse) |
| GET | /api/v1/tasks/ | list_tasks | Defined in task.py (response=TaskListResponse) |
| GET | /api/v1/tasks/{task_id} | get_task_status | Defined in task.py (response=TaskResultResponse) |

## 3. Core Contracts
- POST /api/v1/tasks/ returns HTTP 202 and a non-empty task_id.
- POST /api/v1/jobs/ returns HTTP 202 with job_id and crew_key.
- GET /api/v1/results/{task_id} returns task_id, status, and result.
- GET /api/v1/jobs/{job_id} returns job_id, crew metadata, status, and result.
- Unknown task_id returns HTTP 404 with detail message.
- Validation errors return HTTP 422 for malformed request body.

## 4. Endpoint Contracts
### POST /api/v1/admin
- Handler: create_provider
- Response model: LLMProviderResponse
- Success status: 201

### GET /api/v1/admin
- Handler: list_providers
- Response model: N/A
- Success status: framework default

### GET /api/v1/admin/{name}
- Handler: get_provider
- Response model: LLMProviderResponse
- Success status: framework default

### PUT /api/v1/admin/{name}
- Handler: update_provider
- Response model: LLMProviderResponse
- Success status: framework default

### DELETE /api/v1/admin/{name}
- Handler: delete_provider
- Response model: N/A
- Success status: 204

### GET /api/v1/admin_llm_check/config
- Handler: get_llm_config
- Response model: LLMConfigResponse
- Success status: framework default

### POST /api/v1/auth/register
- Handler: register
- Response model: RegisterResponse
- Success status: 201

### POST /api/v1/auth/token
- Handler: login
- Response model: N/A
- Success status: framework default

### GET /api/v1/auth/me
- Handler: me
- Response model: MeResponse
- Success status: framework default

### GET /api/v1/crews/
- Handler: list_crews
- Response model: CrewListResponse
- Success status: framework default

### POST /api/v1/jobs/
- Handler: submit_job
- Response model: JobSubmitResponse
- Success status: 202

### GET /api/v1/jobs/
- Handler: list_jobs
- Response model: JobListResponse
- Success status: framework default

### GET /api/v1/jobs/{job_id}
- Handler: get_job_status
- Response model: JobStatusResponse
- Success status: framework default

### GET /api/v1/results/{task_id}
- Handler: get_result
- Response model: TaskResultResponse
- Success status: framework default

### POST /api/v1/tasks/
- Handler: create_task
- Response model: TaskCreateResponse
- Success status: 202

### GET /api/v1/tasks/
- Handler: list_tasks
- Response model: TaskListResponse
- Success status: framework default

### GET /api/v1/tasks/{task_id}
- Handler: get_task_status
- Response model: TaskResultResponse
- Success status: framework default


## 5. Validation Rules
- Request body for task creation must be a JSON object.
- Array or scalar payload for task creation should fail with 422.
- Result endpoint should return 404 for unknown task_id.

## 6. Example Payloads
### Task Create Request Fields
- cv_text, job_description, crew_key

### Task Create Response Fields
- task_id

### Task Result Response Fields
- task_id, status, result

## 7. Change Log
- This file is generated from source routes and may be refreshed by automation.
- Last commit: 75d8683 | 2026-05-11 01:01:14 +0700 | nkokkoyit | chore: ignore all test files
- Files changed in last commit:
- .gitignore
