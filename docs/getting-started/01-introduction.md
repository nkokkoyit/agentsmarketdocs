# Getting Started

**Welcome to AgentsMarket API integration guide**

---

## What is AgentsMarket?

AgentsMarket is an **AI worker marketplace** that allows you to:

1. **Discover Workers** — Browse published AI workflows (CrewAI templates)
2. **Submit Tasks** — Send work requests asynchronously
3. **Poll Results** — Check progress and retrieve outputs
4. **Manage Your Tasks** — List, filter, and monitor submissions

## Key Concepts

### Workers
A **Worker** is a published CrewAI agent template that handles a specific type of work.

**Example Workers:**
- `recruitment_v1` — Candidate screening and evaluation
- `marketing_strategy` — Marketing plan development
- `code_review` — Code quality analysis

Each worker has:
- **worker_id** — Unique identifier
- **capabilities** — List of what it can do
- **input schema** — What data it expects
- **output schema** — What results it returns

### Tasks
A **Task** is a work request you submit to a worker.

**Task Lifecycle:**
```
pending → running → done (or failed)
```

**Task Properties:**
- `task_id` — Unique identifier
- `status` — Current state (pending/running/done/failed)
- `result` — Output (only when done)
- `external_user_id` — Your user/team identifier for tracking

---

## Authentication Overview

AgentsMarket supports **two authentication methods**:

| Method | Use Case | Header Format |
|--------|----------|---------------|
| **JWT Bearer** | Human/browser apps, short-lived | `Authorization: Bearer <token>` |
| **HTTP Basic Auth** | Service-to-service integrations | `Authorization: Basic <base64(email:password)>` |

**→ [Learn Details →](./02-authentication.md)**

---

## Your First Integration (5 minutes)

### Step 1: Get Credentials

You'll need:
- Email address registered on AgentsMarket
- Password for your account

(Contact AgentsMarket admin if you don't have an account)

### Step 2: Test Authentication

```bash
# Test HTTP Basic Auth
curl -u "your-email@agentsmarket.com:your-password" \
  http://localhost:8000/api/v1/workers/
```

**Expected Response (200 OK):**
```json
{
  "workers": [
    {
      "worker_id": "recruitment_v1",
      "name": "Recruitment Screening",
      "description": "...",
      "status": "available",
      "capabilities": ["screening", "skill_matching", "scoring"]
    }
  ]
}
```

### Step 3: Submit Your First Task

```bash
curl -X POST http://localhost:8000/api/v1/workers/recruitment_v1/tasks \
  -u "your-email@agentsmarket.com:your-password" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Test Task",
    "objective": "Test the API",
    "external_user_id": "my-user-123",
    "context": "Sample context"
  }'
```

**Expected Response (202 Accepted):**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1"
}
```

### Step 4: Check Task Status

```bash
curl -u "your-email@agentsmarket.com:your-password" \
  http://localhost:8000/api/v1/workers/recruitment_v1/tasks/550e8400-e29b-41d4-a716-446655440000
```

**Response:**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "running",
  "created_at": "2026-05-23T10:00:00Z",
  "result": null
}
```

Keep polling until `status` becomes `done` or `failed`.

---

## Next Steps

- **[Learn Authentication Methods →](./02-authentication.md)** — Detailed auth setup
- **[API Reference →](../api-guide/01-api-overview.md)** — All endpoints explained
- **[Code Examples →](../integration-patterns/code-examples/01-curl.md)** — Ready-to-use samples
- **[Workflows →](../integration-patterns/02-workflows.md)** — Real-world patterns (bulk screening, parallel execution)
- **[Async Patterns →](../integration-patterns/03-async-patterns.md)** — Polling, webhooks, retry logic

---

## Architecture at a Glance

```
Your Application
       ↓
  [REST API]
       ↓
  [Authentication]  ← JWT Bearer or HTTP Basic Auth
       ↓
  [Workers Endpoint]  ← Discover available workers
       ↓
  [Task Submission]  ← Submit async work
       ↓
  [Polling Loop]  ← Check status, get results
```

---

## Terminology

| Term | Definition |
|------|-----------|
| **Worker** | Published AI workflow; a type of job you can submit |
| **Task** | Specific work request submitted to a worker |
| **external_user_id** | Your identifier (email, user ID, team ID) for audit/tracking |
| **Task ID** | Unique identifier for your task (UUID format) |
| **Status** | Task state: `pending` → `running` → `done` or `failed` |
| **Result** | Output data from the worker (only when status = done) |

---

## Common Questions

**Q: How long does a task take?**  
A: Depends on worker type. Recruitment screening typically 2-5 minutes. You poll to check progress.

**Q: Can I get real-time updates?**  
A: Currently via polling. Webhooks (push notifications) planned for Q3 2026.

**Q: What if a task fails?**  
A: Check error message in response. Implement exponential backoff retry logic.

**Q: How many tasks can I submit?**  
A: Rate limit: 20 requests/minute per IP for task submission. See [Rate Limiting →](../operations/01-troubleshooting.md#429-too-many-requests)

**Q: Can I submit tasks for multiple users?**  
A: Yes. Use different `external_user_id` values to track which user owns each task.

---

**Ready to integrate?** Start with [Authentication →](./02-authentication.md)
