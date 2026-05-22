# CEONOVA AgentsMarket Integration Guide

**AgentsMarket Workers API for CEONOVA**

---

## 📋 Overview

AgentsMarket exposes a **REST API** that allows CEONOVA to:
- **Discover workers** — browse published CrewAI templates as AI workers
- **Submit tasks** — send work to a specific worker for async execution
- **Poll results** — check task status and retrieve outputs
- **Manage tasks** — list, filter, and monitor tasks by CEONova user

This guide is for **CEONOVA internal developers** integrating with AgentsMarket in production.

---

## 🚀 Quick Start

### 1. Authenticate
AgentsMarket supports **two authentication methods** on `/api/v1/workers/` endpoints:

| Method | Use Case |
|--------|----------|
| **JWT Bearer** | Human/browser, short-lived token |
| **HTTP Basic Auth** | Service-to-service (CEONova → AgentsMarket) |

→ **[Learn Authentication →](./getting-started/02-authentication.md)**

### 2. Discover Workers
List all available workers and their capabilities:

```bash
curl -X GET http://localhost:8000/api/v1/workers/ \
  -u "ceonova@agentsmarket.com:password"
```

→ **[API Reference →](./api-guide/01-api-overview.md)**

### 3. Submit a Task
Send work to a worker for async execution:

```bash
curl -X POST http://localhost:8000/api/v1/workers/recruitment_v1/tasks \
  -u "ceonova@agentsmarket.com:password" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Screen Alice Johnson",
    "objective": "Evaluate candidate fit",
    "context": "Senior Engineer role, 5+ years Python",
    "expected_output": "Score and recommendation",
    "external_user_id": "ceonova-user-456"
  }'
```

### 4. Poll Task Status
Check progress and retrieve results:

```bash
curl -X GET http://localhost:8000/api/v1/workers/recruitment_v1/tasks/{task_id} \
  -u "ceonova@agentsmarket.com:password"
```

→ **[Code Examples →](./integration-patterns/code-examples/02-python.md)**

---

## 📚 Documentation Structure

| Section | Purpose |
|---------|---------|
| **[Getting Started](./getting-started/01-introduction.md)** | Overview & authentication setup |
| **[API Guide](./api-guide/01-api-overview.md)** | Complete endpoint reference |
| **[Code Examples](./integration-patterns/code-examples/01-curl.md)** | Ready-to-use samples (cURL, Python, JS) |
| **[Integration Patterns](./integration-patterns/02-workflows.md)** | Real-world use cases & workflows |
| **[Async Patterns](./integration-patterns/03-async-patterns.md)** | Polling, webhooks, retry strategies |
| **[Operations](./operations/01-troubleshooting.md)** | Troubleshooting, monitoring, rate limits |
| **[Architecture](./architecture/01-system-overview.md)** | System design & deployment |

---

## 🔧 Common Workflows

### Bulk Recruitment Screening
Screen multiple candidates in parallel, collect scores, rank results.

→ **[See Bulk Screening Workflow →](./integration-patterns/02-workflows.md#workflow-1-bulk-recruitment-screening)**

### Parallel Worker Execution
Run multiple workers on the same input, correlate results.

→ **[See Parallel Execution →](./integration-patterns/02-workflows.md#workflow-2-parallel-worker-execution)**

### Async Patterns with Polling
Implement exponential backoff, circuit breaker, idempotency.

→ **[See Async Patterns →](./integration-patterns/03-async-patterns.md)**

---

## 🔐 Security & Best Practices

- Always use **HTTPS** in production (not HTTP)
- Store credentials in environment variables, never in code
- Implement **exponential backoff** for retries
- Use **HTTP Basic Auth** for service-to-service (more stable)
- Include unique `external_user_id` for audit trails

→ **[See Troubleshooting Guide →](./operations/01-troubleshooting.md)**

---

## 📞 Support & Feedback

For issues, feature requests, or questions:
- Check the **[Troubleshooting Guide](./operations/01-troubleshooting.md)** first
- Review **[OpenAPI Specification](./reference/openapi.yaml)** for schema details
- Contact AgentsMarket support team

---

**Last Updated:** May 23, 2026  
**API Version:** v1  
**Status:** Production Ready
