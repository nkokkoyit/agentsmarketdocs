# Glossary

**Common terms and definitions for AgentsMarket integration**

---

## A

**API** (Application Programming Interface)
: Technical interface allowing external systems to communicate with AgentsMarket. Includes REST endpoints for task submission, discovery, and status polling.

**Async** (Asynchronous)
: Processing that doesn't block the caller. Task submission returns immediately with a task_id; results retrieved later via polling.

**Authentication**
: Process of verifying user identity. AgentsMarket supports JWT Bearer tokens and HTTP Basic Auth (email:password).

---

## B

**Basic Auth**
: HTTP authentication method using email:password encoded in Base64. Preferred for service-to-service integration (CEONOVA).

**Bearer Token**
: JWT access token obtained via `POST /api/v1/auth/token`. Valid for 30 minutes; requires refresh after expiry.

**Backoff** (Exponential Backoff)
: Retry strategy that gradually increases wait time between attempts (1s → 2s → 4s...). Reduces server load and improves resilience.

---

## C

**CEONova**
: External platform integrating with AgentsMarket via REST API. Submits tasks for AI processing (e.g., recruitment screening).

**Crew** (CrewAI)
: Published AI workflow template using CrewAI framework. Defines agents, tasks, and orchestration logic. Exposed as "Workers" to external partners.

**Circuit Breaker**
: Fault tolerance pattern that stops polling if task exceeds max age. Prevents resource waste on stuck tasks.

---

## D

**Dead-Letter Queue (DLQ)**
: Queue for tasks that failed after max retries. Separate processing stream for investigation and manual intervention.

**Developer**
: Person integrating AgentsMarket into external systems (e.g., CEONOVA developer building the integration).

---

## E

**Endpoint**
: HTTP route exposed by AgentsMarket API. Example: `GET /api/v1/workers/`, `POST /api/v1/workers/{worker_id}/tasks`.

**Error Code**
: HTTP status code indicating request result. Examples: 200 (success), 401 (unauthorized), 429 (rate limited).

**external_user_id**
: CEONOVA user/team identifier stored with each task. Used for filtering, audit trails, and IDOR protection. Stored as `ceonova:{external_user_id}` internally.

---

## H

**HTTPS**
: Secure HTTP protocol. Required in production to encrypt credentials in Basic Auth headers.

---

## I

**IDOR** (Insecure Direct Object Reference)
: Security vulnerability where one user can access another user's data. AgentsMarket prevents via external_user_id ownership verification.

**Idempotency**
: Property of operations that can be safely repeated with same result. Critical for retry logic and webhook handling.

**Integration**
: Technical connection between AgentsMarket and external platform (e.g., CEONOVA).

---

## J

**JWT** (JSON Web Token)
: Standard authentication token format. Bearer token used for human/browser authentication. Expires after 30 minutes.

---

## M

**Message Queue**
: System for storing and delivering messages asynchronously. AgentsMarket uses Redis for task queue.

**Microservice**
: Small, independent service handling one concern. AgentsMarket architecture: API service, Worker service, Database, etc.

---

## P

**Polling**
: Client repeatedly checks resource status (e.g., task status) at intervals. Current AgentsMarket mechanism; webhooks planned for Q3 2026.

**Payload**
: Data sent in request body. Example: `{"title": "...", "objective": "...", "external_user_id": "..."}`

---

## R

**Rate Limit**
: Request quota. AgentsMarket: 20 POST requests/minute per IP for task submission.

**REST** (Representational State Transfer)
: Web API design pattern using HTTP methods (GET, POST, etc.). Standard for web APIs.

**Result**
: Output from completed task. Freeform JSON structure dependent on worker type.

---

## S

**Schema**
: Data structure definition. OpenAPI schema specifies request/response formats for all endpoints.

**Status**
: Current state of a task. Values: `pending` → `running` → `done` or `failed`.

**Stateless**
: Design where each request is independent; no session state required. AgentsMarket API is stateless; state stored in database.

---

## T

**Task**
: Work request submitted to a worker. Lifecycle: pending → running → done (or failed). Identified by task_id (UUID).

**Throughput**
: Number of tasks completed per unit time. Affected by polling frequency and worker capacity.

**Timeout**
: Maximum wait time before giving up. Example: polling timeout after 30 attempts (~2-5 minutes).

---

## W

**Webhook**
: HTTP callback where server pushes updates to client. Planned for AgentsMarket Q3 2026; currently use polling.

**Worker**
: Published CrewAI template exposed as AI service. Identified by worker_id. Examples: `recruitment_v1`, `marketing_strategy`.

**Workflow**
: Sequence of tasks/workers executed for a business purpose. Example: bulk screening (submit 50 tasks → poll all → aggregate results).

---

## U

**UUID** (Universally Unique Identifier)
: Format for task_id and other unique identifiers. Example: `550e8400-e29b-41d4-a716-446655440000`

---

## Related Documentation

- [Getting Started](../getting-started/01-introduction.md) — Quick integration guide
- [API Reference](../api-guide/01-api-overview.md) — Endpoint specifications
- [Troubleshooting](../operations/01-troubleshooting.md) — Error codes and debugging
