# System Architecture

**AgentsMarket platform architecture and design**

---

## Architecture Overview

### Core Design Pattern

AgentsMarket follows a **distributed microservices architecture** optimized for:
- **Scalability** — API and workers can scale independently
- **Resilience** — Failure in one component doesn't cascade
- **Asynchronous Processing** — Long-running AI workflows don't block API
- **AI-First Design** — CrewAI agents as first-class components

### Architectural Principles

```
┌─────────────────────────────────────────────────────────┐
│ 1. Separation of Concerns (SoC)                         │
│    - API handles HTTP requests                          │
│    - Workers handle async/batch processing              │
│    - Agents handle AI/business logic                    │
├─────────────────────────────────────────────────────────┤
│ 2. Stateless Services                                   │
│    - Each component can be restarted without impact     │
│    - State stored in external DB/Cache                 │
├─────────────────────────────────────────────────────────┤
│ 3. Async-First for AI Operations                       │
│    - AI workflows dispatched to workers                │
│    - API doesn't wait for agent completion             │
├─────────────────────────────────────────────────────────┤
│ 4. Clear Boundaries                                     │
│    - API/Workers communicate via queue/API             │
│    - Database as single source of truth                │
│    - Frontend → API only (no direct DB access)        │
└─────────────────────────────────────────────────────────┘
```

---

## C4 Architecture Diagrams

### Level 1: System Context

```
┌──────────────────────────────────────────────────────────┐
│                     AGENTSMARKET SYSTEM                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  External System: CEONOVA                               │
│  └─ Sends task requests via REST API                    │
│  └─ Polls task status                                   │
│                                                          │
│  User: Recruiter/Admin                                  │
│  └─ Uses frontend to monitor tasks                      │
│  └─ Browses published workers                           │
│                                                          │
│  External System: CrewAI Framework                       │
│  └─ Defines agent workflows                             │
│  └─ Executes business logic                             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Level 2: Container Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      AGENTSMARKET                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────┐           │
│  │  Frontend (React)                           │           │
│  │  - Browse workers                           │           │
│  │  - View task status                         │           │
│  │  - REST API client                          │           │
│  └────────────┬────────────────────────────────┘           │
│               │                                            │
│               ↓ HTTP/REST (port 3000)                      │
│                                                            │
│  ┌─────────────────────────────────────────────┐           │
│  │  API Server (FastAPI/Flask)                 │           │
│  │  - Authentication (JWT, Basic Auth)         │           │
│  │  - Workers discovery endpoint                │           │
│  │  - Task submission endpoint                  │           │
│  │  - Task polling endpoint                     │           │
│  │  - (port 8000)                              │           │
│  └────────────┬────────────────────────────────┘           │
│               │                                            │
│      ┌────────┴─────────┐                                 │
│      ↓                  ↓                                 │
│  ┌─────────────┐   ┌──────────────────┐                  │
│  │  PostgreSQL │   │  Redis Queue     │                  │
│  │  Database   │   │  (Task Queue)    │                  │
│  └─────────────┘   └────────┬─────────┘                  │
│                             ↓                            │
│  ┌─────────────────────────────────────────┐             │
│  │  Worker Service (Python)                │             │
│  │  - Process tasks from queue             │             │
│  │  - Execute CrewAI agents                │             │
│  │  - Store results in DB                  │             │
│  └─────────────────────────────────────────┘             │
│                                                            │
└─────────────────────────────────────────────────────────────┘
```

### Level 3: Component Diagram (Backend)

```
API Service
├─ Auth Module (JWT, Basic Auth)
├─ Workers Router
│  ├─ GET /api/v1/workers/
│  └─ GET /api/v1/workers/{worker_id}
├─ Tasks Router
│  ├─ POST /api/v1/workers/{worker_id}/tasks
│  ├─ GET /api/v1/workers/{worker_id}/tasks
│  └─ GET /api/v1/workers/{worker_id}/tasks/{task_id}
└─ Database Access Layer

Worker Service
├─ Queue Consumer (Redis)
├─ Agent Executor (CrewAI)
├─ Result Storage
└─ Error Handler

Database (PostgreSQL)
├─ Users table
├─ Workers table
├─ Tasks table
└─ Results table

Cache (Redis)
├─ Task queue
├─ Session store
└─ Rate limit counters
```

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend** | React, HTML, CSS, JS | User interface |
| **Backend** | Python 3.x, FastAPI/Flask | REST API, request handling |
| **AI Engine** | CrewAI | Agent orchestration, workflows |
| **Queue** | Redis | Async task queue |
| **Database** | PostgreSQL | Persistent state |
| **Containerization** | Docker | Deployment, reproducibility |
| **Orchestration** | Docker Compose | Local dev; Kubernetes for prod |
| **Proxy** | Nginx | Load balancing, SSL termination |

---

## Data Flow

### Task Submission Flow

```
1. CEONOVA API Client
   ↓ (POST /api/v1/workers/{worker_id}/tasks)
   
2. API Server (FastAPI)
   ├─ Authenticate request (Basic Auth / JWT)
   ├─ Validate request body
   ├─ Create Task record in PostgreSQL
   ├─ Publish message to Redis queue
   └─ Return 202 Accepted + task_id
   
3. Redis Queue
   └─ Message: {task_id, worker_id, context, ...}
   
4. Worker Service (Python)
   ├─ Consume message from queue
   ├─ Execute CrewAI agent
   ├─ Collect results
   └─ Store in PostgreSQL
   
5. PostgreSQL
   └─ Update task record with result & status=done
   
6. CEONOVA (Polling)
   ↓ (GET /api/v1/workers/{worker_id}/tasks/{task_id})
   
7. API Server (FastAPI)
   ├─ Authenticate request
   ├─ Fetch task from PostgreSQL
   ├─ Return 200 + task status + result
   └─ CEONOVA processes result
```

### Sequence Diagram

```
CEONOVA        API Server      PostgreSQL     Redis Queue      Worker
   │                │                │            │              │
   │─ Discover ───→│                │            │              │
   │  workers      │─ Query ────────→│            │              │
   │               │← Workers ───────│            │              │
   │← Workers ─────│                │            │              │
   │                │                │            │              │
   │─ Submit ──────→│                │            │              │
   │  task          │─ Create ───────→│            │              │
   │               │← Task ID ──────│            │              │
   │               │─ Publish ──────────────────→│              │
   │               │                │            │─ Consume ───→│
   │← 202 Accepted─│                │            │  Consume     │
   │  (task_id)    │                │            │  Execute     │
   │                │                │            │  CrewAI      │
   │                │                │            │              │─ Run Agent
   │                │                │            │              │
   │ (wait 2-5 min)│                │            │              │
   │                │                │            │    Result    │
   │─ Poll ───────→│                │            │  ←─────────  │
   │  status       │─ Fetch ────────→│            │              │
   │               │← Task+Result ──│            │              │
   │               │─ Update ───────→│            │              │
   │← 200 + result─│                │            │              │
   │  (done,       │                │            │              │
   │   data)       │                │            │              │
```

---

## Deployment Architecture

### Development (Docker Compose)

```
┌─────────────────────────────────┐
│      docker-compose up          │
├─────────────────────────────────┤
│                                 │
│  Frontend                       │
│  ├─ node:18                     │
│  └─ Port 3000                   │
│                                 │
│  API Server                     │
│  ├─ Python:3.x                  │
│  ├─ FastAPI/Flask               │
│  └─ Port 8000                   │
│                                 │
│  PostgreSQL                     │
│  ├─ postgres:14                 │
│  └─ Port 5432                   │
│                                 │
│  Redis                          │
│  ├─ redis:7                     │
│  └─ Port 6379                   │
│                                 │
│  Worker Service                 │
│  ├─ Python:3.x                  │
│  └─ Consuming from Redis        │
│                                 │
└─────────────────────────────────┘
```

### Production (Kubernetes)

```
┌────────────────────────────────────────────────┐
│          Kubernetes Cluster                    │
├────────────────────────────────────────────────┤
│                                                │
│  Ingress                                       │
│  └─ Handle external traffic                    │
│                                                │
│  Services                                      │
│  ├─ API Service (LoadBalancer)                 │
│  ├─ Frontend Service (ClusterIP)               │
│  └─ Worker Service (ClusterIP)                 │
│                                                │
│  Deployments                                   │
│  ├─ API (replicas: 3)                          │
│  ├─ Frontend (replicas: 2)                     │
│  ├─ Worker (replicas: 5, auto-scale)           │
│  └─ Redis (StatefulSet)                        │
│                                                │
│  Persistent Volumes                            │
│  ├─ PostgreSQL data                            │
│  └─ Redis data                                 │
│                                                │
│  ConfigMaps                                    │
│  └─ Application config                         │
│                                                │
└────────────────────────────────────────────────┘
```

---

## Scalability

### Horizontal Scaling

1. **API Service** — Add replicas for request handling
   ```bash
   kubectl scale deployment api --replicas=10
   ```

2. **Worker Service** — Add replicas for task processing
   ```bash
   kubectl scale deployment worker --replicas=50
   ```

3. **Database** — Master-replica setup for read scaling
   ```yaml
   database:
     master: primary-db-pod
     replicas:
       - replica-1
       - replica-2
   ```

4. **Redis** — Cluster mode for queue scaling
   ```yaml
   redis:
     mode: cluster
     nodes: 6  # minimum
   ```

### Bottleneck Analysis

| Component | Bottleneck | Solution |
|-----------|-----------|----------|
| API | Request rate | Add replicas, load balancer |
| Worker | Task execution time | Optimize agent logic, add workers |
| Database | Query load | Read replicas, caching |
| Redis | Queue throughput | Cluster mode, sharding |

---

## Security Considerations

1. **Authentication** — JWT or Basic Auth (HTTPS required)
2. **Authorization** — User-based task filtering via external_user_id
3. **Data Encryption** — TLS for transport, encryption at rest for sensitive data
4. **Rate Limiting** — Prevent abuse (20 req/min per IP)
5. **Secret Management** — Store credentials in secure vaults (AWS Secrets Manager, K8s Secrets)
6. **Audit Logging** — Track all API calls with user IDs

---

## Monitoring & Observability

### Metrics

- Task submission rate
- Task completion rate
- Task average duration
- Worker utilization
- API response time
- Error rate
- Rate limit hits

### Logging

- Request/response logs
- Task execution logs
- Error logs with trace IDs
- Audit logs (who called what)

### Alerting

- High error rate (> 5%)
- Slow task execution (> 10 min average)
- Queue backlog (> 1000 pending)
- Worker failure (> 3 consecutive failures)
- Rate limit abuse (> 5 hits/min)

---

## Next Steps

→ **[Getting Started](../getting-started/01-introduction.md)** — Quick integration guide

→ **[API Reference](../api-guide/01-api-overview.md)** — Endpoint documentation

→ **[Code Examples](../integration-patterns/code-examples/01-curl.md)** — Working samples
