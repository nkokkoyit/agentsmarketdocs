# API Overview

**Complete reference for AgentsMarket workers REST API**

---

## Quick Reference

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/v1/workers/` | List all workers |
| GET | `/api/v1/workers/{worker_id}` | Get worker details |
| POST | `/api/v1/workers/{worker_id}/tasks` | Submit a task |
| GET | `/api/v1/workers/{worker_id}/tasks` | List user's tasks |
| GET | `/api/v1/workers/{worker_id}/tasks/{task_id}` | Poll task status & result |

**Base URL:** `http://localhost:8000` (or your AgentsMarket instance)  
**Authentication:** HTTP Basic Auth or JWT Bearer Token  
**Response Format:** JSON

---

## GET /api/v1/workers/

### Description
List all available workers and their capabilities.

### Parameters
None

### Example Request
```bash
curl -X GET http://localhost:8000/api/v1/workers/ \
  -u "ceonova@agentsmarket.com:password"
```

### Example Response (200 OK)
```json
{
  "workers": [
    {
      "worker_id": "recruitment_v1",
      "name": "Recruitment Screening",
      "description": "Evaluate candidates against job criteria using CrewAI agents",
      "status": "available",
      "capabilities": [
        "screening",
        "skill_matching",
        "scoring",
        "recommendation"
      ]
    },
    {
      "worker_id": "marketing_strategy",
      "name": "Marketing Strategy",
      "description": "Create comprehensive marketing plans and content",
      "status": "available",
      "capabilities": [
        "market_research",
        "competitive_analysis",
        "campaign_planning",
        "content_creation"
      ]
    }
  ]
}
```

### Use Cases
- Discover available workers before integration
- Programmatically determine capabilities
- Verify worker availability

---

## GET /api/v1/workers/{worker_id}

### Description
Get detailed information about a specific worker.

### Parameters
| Name | Type | Required | Example |
|------|------|----------|---------|
| worker_id | string (path) | ✅ | `recruitment_v1` |

### Example Request
```bash
curl -X GET http://localhost:8000/api/v1/workers/recruitment_v1 \
  -u "ceonova@agentsmarket.com:password"
```

### Example Response (200 OK)
```json
{
  "worker_id": "recruitment_v1",
  "name": "Recruitment Screening",
  "description": "Evaluate candidates against job criteria using CrewAI agents",
  "status": "available",
  "capabilities": [
    "screening",
    "skill_matching",
    "scoring",
    "recommendation"
  ]
}
```

### Use Cases
- Verify worker exists and is available
- Display worker details in UI
- Confirm capabilities before submission

---

## POST /api/v1/workers/{worker_id}/tasks

### Description
Submit a new task to a worker for async execution.

### Parameters

**Path:**
| Name | Type | Required | Example |
|------|------|----------|---------|
| worker_id | string | ✅ | `recruitment_v1` |

**Request Body:**
```json
{
  "title": "Screen Alice Johnson for Senior Engineer",
  "objective": "Evaluate candidate fit for role",
  "context": "Job: Senior Backend Engineer, 5+ years Python, Docker, AWS",
  "expected_output": "Candidate score (0-100) and recommendation (PASS/MAYBE/REJECT)",
  "external_user_id": "ceonova-user-456"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `title` | string | ✅ | Task name (visible in logs) |
| `objective` | string | ✅ | What the worker should do |
| `context` | string | ❌ | Background info, constraints, input data |
| `expected_output` | string | ❌ | Description of desired result format |
| `external_user_id` | string | ✅ | CEONova user/team ID for tracking & filtering |

### Example Request
```bash
curl -X POST http://localhost:8000/api/v1/workers/recruitment_v1/tasks \
  -u "ceonova@agentsmarket.com:password" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Screen Alice Johnson",
    "objective": "Evaluate candidate fit",
    "context": "Senior Engineer role, 5+ years required",
    "expected_output": "Score (0-100) and recommendation",
    "external_user_id": "ceonova-user-456"
  }'
```

### Response (202 Accepted)
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1"
}
```

### Response Codes
| Code | Meaning | Next Step |
|------|---------|-----------|
| **202** | Task queued successfully | Poll `task_id` for results |
| **401** | Auth failed | Verify credentials |
| **404** | Worker not found | List workers: `GET /api/v1/workers/` |
| **422** | Invalid request body | Check required fields & types |
| **429** | Rate limited (20 req/min) | Wait & retry with backoff |

### Task Lifecycle
```
Submission (202 Accepted)
    ↓
Status: pending (queued, waiting to start)
    ↓
Status: running (worker executing)
    ↓
Status: done (completed with result) or failed (error)
```

### Use Cases
- Submit candidate for screening
- Send document for analysis
- Trigger report generation
- Execute any worker-specific task

---

## GET /api/v1/workers/{worker_id}/tasks

### Description
List all tasks submitted by a specific CEONova user.

### Parameters

**Path:**
| Name | Type | Required | Example |
|------|------|----------|---------|
| worker_id | string | ✅ | `recruitment_v1` |

**Query String:**
| Name | Type | Required | Default | Notes |
|------|------|----------|---------|-------|
| `external_user_id` | string | ✅ | — | CEONova user ID (without `ceonova:` prefix) |
| `limit` | integer | ❌ | 20 | Max results per page (1-100) |
| `offset` | integer | ❌ | 0 | Pagination offset |

### Example Request
```bash
# List first 20 tasks for user 'ceonova-user-456'
curl -X GET "http://localhost:8000/api/v1/workers/recruitment_v1/tasks?external_user_id=ceonova-user-456&limit=20&offset=0" \
  -u "ceonova@agentsmarket.com:password"
```

### Example Response (200 OK)
```json
{
  "total": 52,
  "items": [
    {
      "task_id": "550e8400-e29b-41d4-a716-446655440000",
      "worker_id": "recruitment_v1",
      "status": "done",
      "created_at": "2026-05-23T10:30:00Z"
    },
    {
      "task_id": "660e8400-e29b-41d4-a716-446655440001",
      "worker_id": "recruitment_v1",
      "status": "running",
      "created_at": "2026-05-23T10:35:00Z"
    }
  ]
}
```

### Use Cases
- List user's submitted tasks
- Monitor task queue
- Implement pagination for dashboards
- Find task IDs for polling

---

## GET /api/v1/workers/{worker_id}/tasks/{task_id}

### Description
Poll task status and retrieve results.

### Parameters

**Path:**
| Name | Type | Required | Example |
|------|------|----------|---------|
| worker_id | string | ✅ | `recruitment_v1` |
| task_id | string | ✅ | `550e8400-e29b-41d4-a716-446655440000` |

**Query String (optional):**
| Name | Type | Notes |
|------|------|-------|
| `external_user_id` | string | CEONova user ID (for ownership verification) |

### Example Request
```bash
curl -X GET "http://localhost:8000/api/v1/workers/recruitment_v1/tasks/550e8400-e29b-41d4-a716-446655440000" \
  -u "ceonova@agentsmarket.com:password"
```

### Response Examples

**Status: pending (still queued)**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "pending",
  "result": null
}
```

**Status: running (in progress)**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "running",
  "result": null
}
```

**Status: done (completed successfully)**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "done",
  "result": {
    "overall_score": 85,
    "decision": "PASS",
    "reasoning": "Exceeds all technical requirements",
    "highlights": [
      "7 years experience (exceeds 5+ requirement)",
      "Bonus skills: Kubernetes, CI/CD"
    ],
    "concerns": []
  }
}
```

**Status: failed (error or timeout)**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "failed",
  "result": {
    "error": "Worker timeout after 10 minutes",
    "task_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Task Status Values
| Status | Meaning | Result | Next Action |
|--------|---------|--------|-------------|
| `pending` | Queued, waiting | None | Poll again in 2-5s |
| `running` | Executing | None | Poll again in 2-5s |
| `done` | Success | ✅ Available | Extract `result` |
| `failed` | Error/timeout | ⚠️ Error info | Retry or escalate |
| `unknown` | Not found | None | Check task_id |

### Use Cases
- Poll task progress (typical: 2-5 min per task)
- Retrieve results when complete
- Monitor long-running operations
- Implement retry logic on failure

---

## Error Handling

### 401 Unauthorized
```json
{
  "detail": "Invalid credentials"
}
```
**Fixes:**
- Verify email spelling (case-sensitive)
- Check password correctness
- Ensure HTTPS in production
- Refresh Bearer token if expired

### 404 Not Found
```json
{
  "detail": "Worker not found"
}
```
**Fixes:**
- Verify worker_id matches `GET /api/v1/workers/` response
- Check for typos in worker_id
- Contact support if worker was archived

### 422 Unprocessable Entity
```json
{
  "detail": "Value error, external_user_id is required"
}
```
**Fixes:**
- Ensure all required fields are present
- Verify JSON syntax (valid JSON, correct types)
- Check field names match schema

### 429 Too Many Requests
```json
{
  "detail": "Rate limit exceeded"
}
```
**Rate Limits:**
- POST `/api/v1/workers/{worker_id}/tasks`: **20 requests/minute** per IP
- GET endpoints: Not rate-limited

**Fixes:**
- Implement exponential backoff (1s, 2s, 4s, ...)
- Batch submissions if submitting many tasks
- Contact support for higher limits

---

## Data Types & Schemas

### WorkerInfo
```json
{
  "worker_id": "string",
  "name": "string",
  "description": "string",
  "status": "available",
  "capabilities": ["string"]
}
```

### WorkerTaskSubmitRequest
```json
{
  "title": "string",
  "objective": "string",
  "context": "string (optional)",
  "expected_output": "string (optional)",
  "external_user_id": "string"
}
```

### WorkerTaskStatusResponse
```json
{
  "task_id": "string (UUID)",
  "worker_id": "string",
  "status": "pending | running | done | failed | unknown",
  "result": "any (freeform JSON, varies by worker)"
}
```

---

## What's Next?

→ **[Code Examples](../integration-patterns/code-examples/01-curl.md)** — Ready-to-use samples

→ **[Workflows](../integration-patterns/02-workflows.md)** — Real-world patterns

→ **[Troubleshooting](../operations/01-troubleshooting.md)** — Debug common issues
