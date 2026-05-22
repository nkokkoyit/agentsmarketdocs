# Project Architecture Blueprint: AI Recruitment System

**Generated**: April 30, 2026  
**Project Type**: Python + JavaScript (Microservices)  
**Architecture Pattern**: Microservices + Event-Driven  
**Technology Stack**: Python (FastAPI/Flask), CrewAI, Docker, Redis, PostgreSQL, React  
**Status**: Compliance & Audit Ready

---

## 1. Architecture Detection and Analysis

### Technology Stacks Identified
- **Backend API**: Python 3.x with FastAPI/Flask
- **AI Orchestration**: CrewAI (agent framework)
- **Workers/Queue**: Python workers with async/queue system
- **Frontend**: JavaScript/React (HTML, CSS, JS)
- **Containerization**: Docker (docker-compose)
- **Infrastructure**: Docker + Nginx (proxy)
- **Database**: PostgreSQL (inferred from code structure)

### Architectural Pattern: **Microservices + Event-Driven**
- **API Service** (stateless REST endpoints)
- **Worker Service** (background job processing)
- **Frontend Service** (React SPA)
- **Agent Orchestration Layer** (CrewAI workflows)
- **Queue/Message System** (async communication)

---

## 2. Architectural Overview

### Core Principle
The AI Recruitment System follows a **distributed microservices architecture** optimized for:
1. **Scalability**: API and Workers can scale independently
2. **Resilience**: Failure in one component doesn't cascade
3. **Asynchronous Processing**: Long-running AI workflows don't block API
4. **AI-First Design**: CrewAI agents as first-class components

### Guiding Architectural Principles
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
│    - Frontend -> API only (no direct DB access)        │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Architecture Visualization

### C4 Context Diagram (Level 1)
```
┌──────────────────────────────────────────────────────────────┐
│                     AI RECRUITMENT SYSTEM                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         User/Recruiter Browser                       │   │
│  │  (Frontend: React App running in browser)            │   │
│  └────────────────┬────────────────────────────────────┘   │
│                   │                                        │
│                   ↓ HTTP/REST                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         API Server (Python FastAPI)                  │   │
│  │  - Handle recruitment workflows                      │   │
│  │  - Dispatch agent jobs to queue                      │   │
│  │  - Return results to frontend                        │   │
│  └────────────┬──────────────────────┬─────────────────┘   │
│               │                      │                     │
│         Query/Write           Enqueue Job                  │
│               │                      │                     │
│               ↓                      ↓                     │
│  ┌──────────────────────┐  ┌──────────────────────┐       │
│  │   PostgreSQL DB      │  │  Redis Queue/Cache   │       │
│  │  (Jobs, Results,     │  │  (Job queue, status) │       │
│  │   Candidates)        │  └──────────┬───────────┘       │
│  └──────────────────────┘             │                    │
│                                  Dequeue Job               │
│                                       │                    │
│                                       ↓                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │     Worker Service (Python + CrewAI)                │   │
│  │  - Process recruitment jobs                         │   │
│  │  - Run AI agents (strategy, scoring, etc.)          │   │
│  │  - Write results back to DB                         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Container Diagram (Level 2)
```
┌─────────────────────────────────────────────────────┐
│             Docker Compose Stack                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Frontend Container (Dockerfile.frontend)    │   │
│  │  ┌─────────────────────────────────────┐    │   │
│  │  │ React App (app.js, index.html,      │    │   │
│  │  │ styles.css)                         │    │   │
│  │  └─────────────────────────────────────┘    │   │
│  └────────────┬─────────────────────────────────┘   │
│               │                                     │
│               ↓ HTTP                               │
│  ┌─────────────────────────────────────────────┐   │
│  │  Nginx (nginx.frontend.conf)                 │   │
│  │  - Reverse proxy                             │   │
│  │  - Static file serving                       │   │
│  └────────────┬─────────────────────────────────┘   │
│               │                                     │
│               ↓ HTTP                               │
│  ┌─────────────────────────────────────────────┐   │
│  │  API Container (Dockerfile.api)              │   │
│  │  ┌──────────────────────────────────────┐   │   │
│  │  │ Python API (app/main.py, api/v1)     │   │   │
│  │  │ - Orchestrators (business logic)     │   │   │
│  │  │ - Services (domain services)         │   │   │
│  │  │ - Schemas (request/response models)  │   │   │
│  │  └──────────────────────────────────────┘   │   │
│  └────────────┬────────────┬────────────────────┘   │
│               │            │                       │
│        Query/Write    Enqueue Job                  │
│               │            │                       │
│               ↓            ↓                       │
│  ┌──────────────────┐ ┌──────────────────┐        │
│  │ PostgreSQL DB    │ │ Redis Queue      │        │
│  │ Container        │ │ Container        │        │
│  └──────────────────┘ └────────┬─────────┘        │
│                           Dequeue                 │
│                                │                  │
│                                ↓                  │
│  ┌─────────────────────────────────────────────┐   │
│  │  Worker Container (Dockerfile.worker)       │   │
│  │  ┌──────────────────────────────────────┐   │   │
│  │  │ Python Worker (workers/worker.py)    │   │   │
│  │  │ - CrewAI Orchestration               │   │   │
│  │  │ - Agent execution                    │   │   │
│  │  │ - Result persistence                 │   │   │
│  │  └──────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 4. Core Architectural Components

### 4.1 Frontend Component
**Location**: `frontend/`

| Aspect | Details |
|--------|---------|
| **Purpose** | Provide user interface for recruitment managers/recruiters |
| **Responsibility** | Display candidates, manage workflows, trigger AI analysis |
| **Technology** | React, JavaScript, CSS |
| **Communication** | HTTP/REST to API Server |
| **State Management** | Client-side (React state/localStorage) |
| **Boundaries** | Browser sandbox - cannot access DB or Worker directly |

**Internal Structure**:
```
frontend/
├── index.html           # Entry point
├── app.js              # React application root
├── styles.css          # Global styling
└── [components]        # React components (inferred)
```

### 4.2 API Service Component
**Location**: `app/main.py`

| Aspect | Details |
|--------|---------|
| **Purpose** | Handle HTTP requests, orchestrate business logic, dispatch async jobs |
| **Responsibility** | Request routing, validation, authorization, job dispatch |
| **Technology** | Python (FastAPI/Flask), Uvicorn |
| **Communication** | HTTP/REST (inbound), Redis Queue (outbound) |
| **Scalability** | Stateless - can run multiple instances behind load balancer |
| **Dependencies** | PostgreSQL, Redis, Pydantic (validation) |

**Internal Structure**:
```
app/
├── main.py                    # Application entry point
├── __init__.py
├── api/
│   ├── v1/                    # API v1 endpoints
│   │   ├── jobs.py           # Job endpoints
│   │   ├── candidates.py      # Candidate endpoints
│   │   └── results.py         # Results endpoints
├── application/
│   ├── orchestrators/         # Business logic orchestration
│   │   ├── recruitment_orchestrator.py
│   │   └── candidate_orchestrator.py
│   └── services/              # Domain services
│       ├── job_service.py
│       ├── candidate_service.py
│       └── agent_service.py
├── core/
│   ├── config.py              # Configuration management
│   ├── security.py            # Auth/authz
│   ├── logging.py             # Logging setup
│   └── cache.py               # Cache management
├── domain/
│   ├── models/                # Domain entities
│   │   ├── job.py
│   │   ├── candidate.py
│   │   └── result.py
│   └── services/              # Domain service interfaces
├── infrastructure/
│   ├── db/                    # Database layer
│   ├── external/              # External service integrations
│   ├── queue/                 # Queue management
│   └── workflow/              # Workflow execution
├── schemas/                   # Pydantic request/response models
│   ├── job.py
│   ├── candidate.py
│   ├── crew.py
│   └── result.py
└── workers/                   # Background task definitions
    └── worker.py
```

### 4.3 Worker Service Component
**Location**: `workers/worker.py`

| Aspect | Details |
|--------|---------|
| **Purpose** | Execute long-running AI workflows asynchronously |
| **Responsibility** | Dequeue jobs, run CrewAI agents, persist results |
| **Technology** | Python, CrewAI, async/await |
| **Communication** | Redis Queue (inbound), PostgreSQL (outbound) |
| **Scalability** | Stateless - can run multiple worker instances |
| **Failure Handling** | Job retry logic, dead-letter queue support |

**Processing Flow**:
```
┌──────────────────────────────────────────┐
│ 1. Dequeue Job from Redis                │
│    (Job type: "recruitment_analysis")    │
└────────────┬─────────────────────────────┘
             │
             ↓
┌──────────────────────────────────────────┐
│ 2. Load Job Parameters                   │
│    (candidates, criteria, context)       │
└────────────┬─────────────────────────────┘
             │
             ↓
┌──────────────────────────────────────────┐
│ 3. Initialize CrewAI Agents              │
│    (recruiters_strategy, scoring_agent)  │
└────────────┬─────────────────────────────┘
             │
             ↓
┌──────────────────────────────────────────┐
│ 4. Execute Agent Workflow                │
│    (strategy → scoring → final decision) │
└────────────┬─────────────────────────────┘
             │
             ↓
┌──────────────────────────────────────────┐
│ 5. Persist Results to DB                 │
│    (scores, decisions, explanations)     │
└────────────┬─────────────────────────────┘
             │
             ↓
┌──────────────────────────────────────────┐
│ 6. Update Job Status in Cache            │
│    (COMPLETED, ready for frontend)       │
└──────────────────────────────────────────┘
```

### 4.4 CrewAI Agent Orchestration Component
**Location**: `workflows/` + `infrastructure/workflow/`

| Aspect | Details |
|--------|---------|
| **Purpose** | Define and orchestrate AI agents for recruitment tasks |
| **Responsibility** | Agent role definition, task orchestration, result aggregation |
| **Technology** | CrewAI framework, YAML configs |
| **Communication** | Internal (agent-to-agent) + persistent storage (results) |
| **Agents Identified** | `marketing_strategy_v1.yaml`, `recruitment_v1.yaml`, `scoring_only_v1.yaml` |

**Workflow Types**:
```
1. recruitment_v1.yaml
   └── Full recruitment workflow
       ├── Agent: Recruiters Strategy
       │   └── Define candidate evaluation criteria
       ├── Agent: Scoring Agent
       │   └── Score candidates
       └── Agent: Final Decision Agent
           └── Make final recommendations

2. marketing_strategy_v1.yaml
   └── Marketing-focused recruitment
       └── [Agents TBD]

3. scoring_only_v1.yaml
   └── Lightweight scoring only
       └── Agent: Scoring Agent
           └── Score candidates
```

### 4.5 Data Layer
**Location**: Database + Cache

#### PostgreSQL Database
```sql
-- Core entities (inferred from code structure)
CREATE TABLE jobs (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    description TEXT,
    criteria JSONB,
    created_at TIMESTAMP,
    status VARCHAR(50)
);

CREATE TABLE candidates (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    resume JSONB,
    job_id INTEGER REFERENCES jobs(id),
    created_at TIMESTAMP
);

CREATE TABLE results (
    id SERIAL PRIMARY KEY,
    job_id INTEGER REFERENCES jobs(id),
    candidate_id INTEGER REFERENCES candidates(id),
    score DECIMAL(5,2),
    decision VARCHAR(100),
    explanation TEXT,
    agent_output JSONB,
    created_at TIMESTAMP
);

CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    job_id INTEGER REFERENCES jobs(id),
    task_type VARCHAR(100),
    status VARCHAR(50),
    priority INTEGER,
    created_at TIMESTAMP
);
```

#### Redis Cache
```
KEY PATTERNS:
- job:{job_id}:status          # Job processing status
- job:{job_id}:queue_position  # Queue position
- queue:pending_jobs           # List of pending jobs
- cache:candidates:{job_id}    # Cached candidate data
- cache:results:{job_id}       # Cached results
```

---

## 5. Architectural Layers and Dependencies

### Layer Structure
```
┌─────────────────────────────────────────────────────┐
│                  PRESENTATION LAYER                 │
│         (Frontend: React SPA via Browser)            │
├─────────────────────────────────────────────────────┤
│                  API LAYER                          │
│         (HTTP endpoints, request validation)         │
├─────────────────────────────────────────────────────┤
│              ORCHESTRATION LAYER                    │
│      (Business logic, workflow coordination)         │
├─────────────────────────────────────────────────────┤
│              APPLICATION SERVICES LAYER             │
│      (Domain services, business rules)               │
├─────────────────────────────────────────────────────┤
│              AGENT ORCHESTRATION LAYER              │
│   (CrewAI agents, task definitions, workflows)      │
├─────────────────────────────────────────────────────┤
│            INFRASTRUCTURE LAYER                     │
│    (Database, cache, queue, external APIs)          │
└─────────────────────────────────────────────────────┘
```

### Dependency Rules
```
✅ ALLOWED:
- Presentation → API Layer
- API → Orchestration
- Orchestration → Services
- Services → Domain Model
- Orchestration → Infrastructure (via dependency injection)

❌ NOT ALLOWED:
- Frontend → Database (direct)
- API → Presentation
- Services → API (circular)
- Presentation → Infrastructure (direct)

⚠️  ASYNC BOUNDARIES:
- API → Workers (via Redis queue, decoupled)
- Workers → Database (write results)
- Workers → Cache (update job status)
```

### Dependency Injection Pattern
```python
# Orchestrator receives dependencies
class RecruitmentOrchestrator:
    def __init__(
        self,
        candidate_service: CandidateService,      # Injected
        job_service: JobService,                  # Injected
        queue_manager: QueueManager,              # Injected
        agent_orchestrator: CrewAIOrchestrator   # Injected
    ):
        self.candidate_service = candidate_service
        self.job_service = job_service
        self.queue_manager = queue_manager
        self.agent_orchestrator = agent_orchestrator
    
    async def analyze_candidates(self, job_id):
        # Use injected dependencies
        candidates = await self.candidate_service.get_by_job(job_id)
        job = await self.job_service.get(job_id)
        
        # Dispatch to worker
        job_data = {
            "job_id": job_id,
            "candidates": candidates,
            "criteria": job.criteria
        }
        await self.queue_manager.enqueue("recruitment_analysis", job_data)
```

---

## 6. Data Architecture

### Domain Model
```
Job
├── id (PK)
├── title
├── description
├── criteria (JSONB)
├── status (ENUM: DRAFT, OPEN, IN_PROGRESS, CLOSED)
├── created_at
└── updated_at

Candidate
├── id (PK)
├── job_id (FK → Job)
├── name
├── resume (JSONB)
├── source
├── created_at
└── updated_at

Result
├── id (PK)
├── job_id (FK → Job)
├── candidate_id (FK → Candidate)
├── score (0-100)
├── decision (ENUM: PASS, REJECT, MAYBE, PENDING)
├── explanation (TEXT)
├── agent_output (JSONB)
├── created_at
└── updated_at

Task
├── id (PK)
├── job_id (FK → Job)
├── task_type (ENUM: ANALYSIS, SCORING, DECISION)
├── status (ENUM: PENDING, RUNNING, COMPLETED, FAILED)
├── priority (1-10)
├── retry_count
└── created_at
```

### Entity Relationships
```
Job 1─────N Candidate
 │           │
 │           │
 └─────N─────┴──→ Result
```

### Data Access Patterns
```python
# Repository Pattern
class CandidateRepository:
    async def get_by_job(self, job_id: int) -> List[Candidate]:
        """Get all candidates for a job"""
        query = "SELECT * FROM candidates WHERE job_id = $1"
        return await self.db.fetch(query, job_id)
    
    async def upsert(self, candidate: Candidate) -> Candidate:
        """Create or update candidate"""
        # Upsert logic
        pass

class ResultRepository:
    async def bulk_insert(self, results: List[Result]) -> None:
        """Bulk insert results from agent processing"""
        # Batch insert
        pass
```

### Data Transformation
```
AGENT OUTPUT (Raw JSON)
↓
TRANSFORMATION LAYER
├── Validate format
├── Normalize scores (0-100)
├── Map decisions (PASS/REJECT/MAYBE)
└── Extract explanations
↓
RESULT ENTITY (Structured)
```

### Caching Strategy
```
CACHE-ASIDE PATTERN:
1. Check Redis cache for job status
2. If miss, query database
3. Write result to cache with TTL
4. Return to consumer

INVALIDATION:
- Job status cache: 5 minute TTL
- Candidate list: Invalidate on job update
- Results: Invalidate when new score added
```

---

## 7. Cross-Cutting Concerns Implementation

### 7.1 Authentication & Authorization

**Location**: `core/security.py`

```python
# JWT Token-based auth
class SecurityManager:
    def verify_token(self, token: str) -> User:
        """Verify JWT token from Authorization header"""
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
            return User(**payload)
        except jwt.InvalidTokenError:
            raise UnauthorizedError()
    
    def has_permission(self, user: User, resource: str, action: str) -> bool:
        """Check RBAC permissions"""
        return user.role in PERMISSIONS[resource][action]

# Usage in API
@app.get("/jobs/{job_id}")
async def get_job(job_id: int, user: User = Depends(get_current_user)):
    if not has_permission(user, "jobs", "read"):
        raise ForbiddenError()
    return job_service.get(job_id)
```

**Permissions Model**:
```
ROLES:
- admin: Full access
- recruiter: Read jobs, candidates; create analyses
- viewer: Read-only access

RESOURCES:
- jobs: create, read, update, delete
- candidates: create, read, update
- results: read, export
- workflows: list, execute
```

### 7.2 Error Handling & Resilience

**Location**: `core/logging.py`, `infrastructure/`

```python
# Hierarchical exception handling
class RecruitmentError(Exception):
    """Base exception for recruitment domain"""
    pass

class CandidateNotFoundError(RecruitmentError):
    """Thrown when candidate doesn't exist"""
    pass

class AgentExecutionError(RecruitmentError):
    """Thrown when agent fails to process"""
    pass

# Retry logic
@retry(
    max_attempts=3,
    backoff_factor=2,
    retriable_exceptions=[AgentExecutionError, DatabaseError]
)
async def execute_agent_workflow(job_data):
    """Execute workflow with automatic retries"""
    return await agent_orchestrator.run(job_data)

# Fallback strategy
async def analyze_candidates_with_fallback(job_id):
    try:
        return await agent_orchestrator.full_analysis(job_id)
    except AgentExecutionError as e:
        logger.warning(f"Full analysis failed: {e}")
        # Fallback: use basic scoring only
        return await agent_orchestrator.score_only(job_id)
```

### 7.3 Logging & Monitoring

**Location**: `core/logging.py`

```python
# Structured logging
logger = structlog.get_logger(__name__)

# Log levels by component
logger.info("job_started", job_id=123, user_id=456)
logger.info("agent_executing", agent="recruiters_strategy", job_id=123)
logger.warning("agent_slow", agent="scoring", duration_ms=5000, threshold=2000)
logger.error("agent_failed", agent="final_decision", error=str(e), job_id=123)

# Metrics
metrics.counter("job.submitted", tags={"workflow": "full_recruitment"})
metrics.gauge("queue.size", value=42)
metrics.histogram("agent.execution_time", value=2500, tags={"agent": "scoring"})
```

### 7.4 Input Validation

**Location**: `schemas/`, `core/security.py`

```python
# Pydantic validation
class JobAnalysisRequest(BaseModel):
    job_id: int
    candidate_ids: List[int] = Field(..., min_items=1, max_items=1000)
    criteria: Dict[str, Any]
    
    @validator('candidate_ids')
    def validate_candidate_ids(cls, v):
        if len(v) > 1000:
            raise ValueError('Maximum 1000 candidates per analysis')
        return v

# API endpoint uses automatic validation
@app.post("/jobs/{job_id}/analyze")
async def analyze_job(
    job_id: int,
    request: JobAnalysisRequest,  # Validates automatically
    user: User = Depends(get_current_user)
):
    # request is guaranteed to be valid
    pass
```

### 7.5 Configuration Management

**Location**: `core/config.py`

```python
# Environment-based config
class Settings(BaseSettings):
    # Database
    db_host: str = os.getenv("DB_HOST", "localhost")
    db_port: int = int(os.getenv("DB_PORT", 5432))
    db_user: str = os.getenv("DB_USER")
    db_password: str  # From env only (required)
    
    # Redis
    redis_url: str = os.getenv("REDIS_URL", "redis://localhost")
    
    # CrewAI
    agent_timeout: int = int(os.getenv("AGENT_TIMEOUT", 300))
    agent_retries: int = int(os.getenv("AGENT_RETRIES", 3))
    
    # Security
    jwt_secret: str  # From env only (required)
    jwt_algorithm: str = "HS256"
    jwt_expiry: int = 3600
    
    # Environment
    environment: str = os.getenv("ENVIRONMENT", "development")
    debug: bool = environment == "development"
    
    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 8. Service Communication Patterns

### 8.1 Synchronous Communication (API)

**Frontend ↔ API**:
```
REQUEST:
POST /api/v1/jobs/123/analyze
Content-Type: application/json
Authorization: Bearer {token}

{
  "candidate_ids": [1, 2, 3],
  "criteria": { "min_experience": 5 }
}

RESPONSE:
{
  "job_id": 123,
  "job_status": "PROCESSING",
  "queue_position": 5,
  "estimated_wait": "2 minutes"
}
```

**API ↔ Database**:
```
Connection pooling (sqlalchemy):
- Min connections: 5
- Max connections: 20
- Connection timeout: 5s
- Query timeout: 30s
```

### 8.2 Asynchronous Communication (Queue)

**API → Worker (via Redis)**:
```
MESSAGE FORMAT:
{
  "job_id": "job_123",
  "job_type": "recruitment_analysis",
  "payload": {
    "job_id": 123,
    "candidates": [...],
    "criteria": {...}
  },
  "timestamp": "2026-04-30T10:00:00Z",
  "retry_count": 0,
  "max_retries": 3
}

QUEUE TOPOLOGY:
- primary_queue: recruitment_analysis jobs
- priority_queue: urgent analyses (scored_only)
- dead_letter_queue: failed jobs after max retries
```

**Worker → Database (Results)**:
```
ATOMIC WRITE:
1. Update job status to PROCESSING
2. Bulk insert results
3. Update job status to COMPLETED
4. Update cache with completion time

ALL IN A TRANSACTION (rollback on failure)
```

### 8.3 Agent-to-Agent Communication (Internal)

```
Crew Definition (YAML):
crews:
  - name: recruitment_crew
    agents:
      - id: recruiters_strategy
        role: Recruitment Strategist
        goal: Define evaluation strategy
      
      - id: scoring_agent
        role: Candidate Scorer
        goal: Score candidates
      
      - id: final_decision
        role: Final Decision Maker
        goal: Make recommendations
    
    tasks:
      - name: define_strategy
        agent: recruiters_strategy
        description: Define how to evaluate candidates
      
      - name: score_candidates
        agent: scoring_agent
        description: Score each candidate
        depends_on: define_strategy
      
      - name: make_decision
        agent: final_decision
        description: Make final recommendations
        depends_on: score_candidates
```

---

## 9. Technology-Specific Architectural Patterns

### 9.1 Python Architectural Patterns

**Module Organization**:
```
app/
├── api/             # HTTP layer
├── application/     # Business logic (orchestrators, services)
├── core/           # Cross-cutting concerns
├── domain/         # Domain models
├── infrastructure/ # External integrations
└── schemas/        # Request/response validation
```

**Dependency Injection**:
```python
# FastAPI dependency injection
def get_db() -> Database:
    return Database(settings.db_url)

def get_job_service(db: Database = Depends(get_db)) -> JobService:
    return JobService(db)

@app.get("/jobs/{job_id}")
async def get_job(
    job_id: int,
    service: JobService = Depends(get_job_service)
):
    return await service.get(job_id)
```

**Async/Await**:
```python
# All I/O operations are async
async def analyze_candidates(self, job_id: int):
    candidates = await self.db.fetch_candidates(job_id)
    job = await self.db.fetch_job(job_id)
    
    # Dispatch to worker queue
    await self.queue.enqueue("recruitment_analysis", {
        "job_id": job_id,
        "candidates": candidates
    })
    
    return {"status": "PROCESSING", "queue_position": 5}
```

### 9.2 React Architectural Patterns

**Component Structure** (inferred):
```javascript
// App component - root
<App>
  ├── <Header /> - Navigation, user profile
  ├── <JobList /> - List of jobs
  │   └── <JobCard /> - Job summary (reusable)
  ├── <CandidateList /> - List of candidates for selected job
  │   └── <CandidateCard /> - Candidate info (reusable)
  └── <ResultsPanel /> - Analysis results
      └── <ResultCard /> - Individual result
```

**State Management** (likely):
```javascript
// Component-level state
const [jobs, setJobs] = useState([]);
const [selectedJob, setSelectedJob] = useState(null);
const [results, setResults] = useState([]);

// API communication
useEffect(() => {
    fetchJobs().then(setJobs);
}, []);

const handleAnalyze = async (jobId) => {
    const response = await api.post(`/jobs/${jobId}/analyze`, {...});
    setResults(response.results);
};
```

---

## 10. Implementation Patterns

### 10.1 Orchestrator Pattern

```python
class RecruitmentOrchestrator:
    """Orchestrates recruitment workflow"""
    
    def __init__(self, services, queue_manager):
        self.candidate_service = services.candidate
        self.job_service = services.job
        self.result_service = services.result
        self.queue_manager = queue_manager
    
    async def analyze_candidates(self, job_id: int) -> AnalysisJob:
        # 1. Validate
        job = await self.job_service.get(job_id)
        if not job:
            raise JobNotFoundError(job_id)
        
        # 2. Fetch data
        candidates = await self.candidate_service.get_by_job(job_id)
        if not candidates:
            raise NoCandidatesError(job_id)
        
        # 3. Dispatch async work
        queue_job = await self.queue_manager.enqueue(
            "recruitment_analysis",
            {
                "job_id": job_id,
                "candidates": candidates,
                "criteria": job.criteria
            }
        )
        
        # 4. Return tracking info
        return AnalysisJob(
            id=queue_job.id,
            status="QUEUED",
            position=queue_job.position
        )
```

### 10.2 Repository Pattern

```python
class CandidateRepository:
    """Data access for candidates"""
    
    def __init__(self, db: Database):
        self.db = db
    
    async def get(self, id: int) -> Optional[Candidate]:
        row = await self.db.fetch_one(
            "SELECT * FROM candidates WHERE id = $1",
            id
        )
        return Candidate(**row) if row else None
    
    async def get_by_job(self, job_id: int) -> List[Candidate]:
        rows = await self.db.fetch(
            "SELECT * FROM candidates WHERE job_id = $1",
            job_id
        )
        return [Candidate(**row) for row in rows]
    
    async def save(self, candidate: Candidate) -> Candidate:
        result = await self.db.execute(
            """
            INSERT INTO candidates (name, resume, job_id, created_at)
            VALUES ($1, $2, $3, $4)
            RETURNING *
            """,
            candidate.name,
            candidate.resume,
            candidate.job_id,
            candidate.created_at
        )
        return Candidate(**result)
```

### 10.3 Service Pattern

```python
class JobService:
    """Business logic for job management"""
    
    def __init__(self, repository: JobRepository, logger):
        self.repository = repository
        self.logger = logger
    
    async def create(self, job_data: JobCreateRequest) -> Job:
        # Validate business rules
        if not job_data.title:
            raise ValidationError("Job title required")
        
        # Create entity
        job = Job(
            title=job_data.title,
            description=job_data.description,
            criteria=job_data.criteria,
            status="DRAFT",
            created_at=datetime.utcnow()
        )
        
        # Persist
        saved_job = await self.repository.save(job)
        
        # Log
        self.logger.info("job_created", job_id=saved_job.id)
        
        return saved_job
```

---

## 11. Testing Architecture

### Test Pyramid

```
         /\
        /  \          E2E Tests (5%)
       /────\         - Full system integration
      /      \        - Frontend → API → DB
     /        \
    /──────────\      Integration Tests (20%)
   /            \     - API + Database
  /              \    - Queue system
 /                \
/──────────────────\ Unit Tests (75%)
                    - Services
                    - Orchestrators
                    - Data transformations
```

### Test Files

```
tests/
├── run_agent_role_contract_test.py   # Agent contract tests
├── run_agent_test.py                 # Agent unit tests
├── run_api_contract_test.py         # API contract tests
├── run_crewai_real_test.py          # Real agent execution tests
├── test_crew_picker_modal.py        # Frontend modal tests
├── TEST_REPORT_TEMPLATE.md          # Test reporting standard
└── result/                          # Test artifacts
    ├── agent_role_contract_test_*.json
    └── agent_role_contract_test_*.md
```

### Testing Patterns

```python
# Unit Test Example
@pytest.mark.asyncio
async def test_candidate_service_get():
    """Test retrieving candidate"""
    # Arrange
    mock_repo = MagicMock()
    service = CandidateService(mock_repo)
    expected = Candidate(id=1, name="John Doe")
    mock_repo.get.return_value = expected
    
    # Act
    result = await service.get(1)
    
    # Assert
    assert result.id == 1
    assert result.name == "John Doe"
    mock_repo.get.assert_called_once_with(1)

# Integration Test Example
@pytest.mark.asyncio
async def test_recruitment_orchestrator_analyze():
    """Test full analysis workflow"""
    # Use real database in test container
    db = await get_test_db()
    orchestrator = RecruitmentOrchestrator(db)
    
    # Create test data
    job = await db.create_job(title="Software Engineer")
    candidates = await db.create_candidates(job.id, count=3)
    
    # Execute
    result = await orchestrator.analyze_candidates(job.id)
    
    # Assert
    assert result.status == "QUEUED"
```

---

## 12. Deployment Architecture

### Container Images

| Container | Base Image | Size | Purpose |
|-----------|-----------|------|---------|
| API | `python:3.11-slim` | ~200MB | REST API server |
| Worker | `python:3.11-slim` | ~250MB | AI agent execution |
| Frontend | `node:20-alpine` | ~100MB | React SPA |
| Database | `postgres:15-alpine` | ~150MB | Data persistence |
| Redis | `redis:7-alpine` | ~30MB | Queue & cache |
| Nginx | `nginx:alpine` | ~40MB | Reverse proxy |

### Docker Compose Services

```yaml
services:
  api:
    image: recruitment-api:latest
    ports: ["8000:8000"]
    environment:
      - DB_HOST=postgres
      - DB_USER=recruitment
      - REDIS_URL=redis://redis:6379
    depends_on: [postgres, redis]

  worker:
    image: recruitment-worker:latest
    environment:
      - DB_HOST=postgres
      - REDIS_URL=redis://redis:6379
    depends_on: [postgres, redis]
    command: python -m workers.worker

  frontend:
    image: recruitment-frontend:latest
    ports: ["80:80"]
    depends_on: [api]

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: recruitment
      POSTGRES_USER: recruitment
    volumes: [postgres_data:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

### Environment Configuration

```
DEVELOPMENT:
- DEBUG=true
- LOG_LEVEL=DEBUG
- DB_POOL_SIZE=5
- AGENT_TIMEOUT=300

PRODUCTION:
- DEBUG=false
- LOG_LEVEL=INFO
- DB_POOL_SIZE=20
- AGENT_TIMEOUT=600
- ENABLE_MONITORING=true
- ENABLE_SECURITY_CHECKS=true
```

### Scaling Considerations

```
HORIZONTAL SCALING:
├── API: Scale instances behind load balancer
├── Worker: Scale pool by job queue depth
│   - If queue > 100 jobs: scale to 5 workers
│   - If queue < 10 jobs: scale to 1 worker
└── Frontend: Serve from CDN

VERTICAL SCALING:
├── Database: Add CPU/RAM as job volume grows
├── Redis: Add RAM for larger queue depths
└── API/Worker: Add CPU for parallel processing
```

---

## 13. Extensibility & Evolution Patterns

### Adding New Agents

```yaml
# workflows/new_agent_v1.yaml
agents:
  - id: new_agent
    role: "New Specialist"
    goal: "Accomplish new task"
    tools:
      - search_tool
      - analysis_tool

tasks:
  - name: task_1
    agent: new_agent
    description: "Execute new task"
```

```python
# In worker.py
class NewAgentOrchestrator(AgentOrchestrator):
    """Orchestrate new agent"""
    
    async def execute(self, job_data):
        crew = self._build_crew("new_agent")
        return await crew.kickoff(job_data)
```

### Adding New API Endpoints

```python
# app/api/v1/new_resource.py
@router.post("/new-resource", response_model=NewResourceResponse)
async def create_resource(
    request: NewResourceRequest,
    service: NewResourceService = Depends(get_service)
):
    return await service.create(request)

# Register in app/main.py
app.include_router(new_resource_router, prefix="/api/v1")
```

### Adding New Data Models

```python
# app/domain/models/new_model.py
class NewModel(BaseModel):
    id: int
    name: str
    # ... fields

# app/infrastructure/db/new_model_repo.py
class NewModelRepository:
    async def save(self, model: NewModel):
        # Implement persistence
        pass
```

---

## 14. Architectural Governance

### Code Organization Standards
- Consistent folder structure across services
- Clear dependency flows (no circular dependencies)
- Documented interfaces and contracts

### Automated Checks
```yaml
# .github/workflows/architecture-check.yml
lint:
  - ESLint (frontend)
  - Pylint (backend)
security:
  - CodeQL scanning
  - Dependency vulnerability checks
  - Secret scanning
testing:
  - Unit tests (>80% coverage)
  - Integration tests
  - Contract tests (API)
quality:
  - SonarQube analysis
  - Type checking (mypy)
```

### Architectural Review Checklist
- [ ] Components follow single responsibility
- [ ] No circular dependencies
- [ ] External dependencies injected
- [ ] Error handling implemented
- [ ] Logging added at boundaries
- [ ] Tests included (unit + integration)
- [ ] Documentation updated

---

## 15. Blueprint for New Development

### Development Workflow

```
1. UNDERSTAND REQUIREMENTS
   └── Map to architecture component

2. DESIGN SOLUTION
   ├── Identify which layer (API, service, agent)
   ├── Define interfaces/contracts
   └── Plan data flow

3. IMPLEMENT
   ├── Write tests first (TDD)
   ├── Implement component
   ├── Add logging & error handling
   └── Update documentation

4. INTEGRATE
   ├── Connect to dependent components
   ├── Register dependencies
   └── Update docker-compose if needed

5. TEST
   ├── Unit tests pass
   ├── Integration tests pass
   └── Manual testing complete

6. DEPLOY
   ├── Build container
   ├── Run preflight checks
   └── Deploy with rollback plan
```

### Implementation Templates

#### New Service Template
```python
# app/application/services/new_service.py
class NewService:
    """Service for managing new resource"""
    
    def __init__(self, repository, logger):
        self.repository = repository
        self.logger = logger
    
    async def create(self, data):
        # Validate
        # Create entity
        # Persist
        # Log
        # Return
        pass
```

#### New API Endpoint Template
```python
# app/api/v1/new_resource.py
@router.post("/resources", response_model=NewResourceResponse)
async def create_resource(
    request: NewResourceRequest,
    service: NewService = Depends(get_service),
    user: User = Depends(get_current_user)
):
    # Validate permissions
    # Call service
    # Return result
    pass
```

### Common Pitfalls to Avoid
- ❌ Direct database calls from API (use services)
- ❌ Blocking operations in async code
- ❌ Hardcoded configuration values
- ❌ Missing error handling
- ❌ Tests that test implementation, not behavior
- ❌ Circular dependencies between modules

---

## 16. Architecture Governance & Compliance

### Compliance Requirements
1. **Data Protection**: All sensitive data encrypted at rest & in transit
2. **Audit Trail**: All operations logged with user attribution
3. **Access Control**: RBAC enforced at API boundaries
4. **Testing**: >80% code coverage, contract tests for APIs
5. **Documentation**: Architecture, APIs, and deployment procedures documented
6. **Security**: Regular security scans (CodeQL, OWASP)

### Governance Artifacts
- [x] Architecture Blueprint (this document)
- [ ] API Specification (OpenAPI/Swagger)
- [ ] Data Flow Diagrams
- [ ] Threat Model Analysis
- [ ] Deployment Procedures
- [ ] Operational Runbooks

### Review Cadence
- **Monthly**: Architecture review for new changes
- **Quarterly**: Full architecture assessment
- **Ad-hoc**: Major refactorings or new integrations

---

## 17. Architecture Maintenance & Evolution

### Version Control
- `git tag` architecture releases (e.g., `architecture-v1.0`)
- Document breaking changes in `ARCHITECTURE_CHANGELOG.md`
- Update blueprint when major changes occur

### Evolution Triggers
- New agent type implemented → Update workflows section
- New service added → Update layers section
- Deployment model changes → Update deployment section
- New cross-cutting concern → Update section 7

### Next Steps
1. **API Specification**: Document all endpoints with OpenAPI
2. **Data Flow Diagrams**: Detailed message flow between components
3. **Threat Modeling**: Identify security risks (STRIDE-A)
4. **Deployment Guide**: Step-by-step production deployment
5. **Operational Runbook**: How to run, monitor, troubleshoot in production

---

**Last Updated**: April 30, 2026  
**Status**: Approved for Compliance & Audit  
**Next Review**: July 30, 2026
