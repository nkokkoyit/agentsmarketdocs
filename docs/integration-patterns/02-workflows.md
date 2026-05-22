# Workflows

**Real-world patterns and use cases for AgentsMarket integration**

---

## Workflow 1: Bulk Recruitment Screening

**Scenario:** Screen 50 candidates for a single job opening in parallel, collect scores, rank results.

### Overview

```
Job Posted
  ↓
Submit 50 tasks in parallel (2-3 sec)
  ↓
Poll all tasks (5-10 min)
  ↓
Aggregate & rank scores
  ↓
Hiring team reviews top 10
```

### Implementation (Python)

**Full code example:**

```python
import requests
import concurrent.futures
import time

API_BASE = "http://localhost:8000"
AUTH = ("ceonova@agentsmarket.com", "password")

# Candidate list
candidates = [
    {"name": "Alice Johnson", "resume": "7 years Python, Docker, AWS, 3 years Kubernetes"},
    {"name": "Bob Smith", "resume": "5 years Python, 1 year Docker"},
    # ... 48 more candidates
]

job_description = "Senior Backend Engineer, 5+ years Python, Docker, AWS required"
external_user_id = "hiring-team-1"

# Step 1: Submit all tasks in parallel
print(f"Submitting {len(candidates)} screening tasks...")
task_ids = []

def submit_candidate(candidate):
    response = requests.post(
        f"{API_BASE}/api/v1/workers/recruitment_v1/tasks",
        json={
            "title": f"Screen {candidate['name']}",
            "objective": "Evaluate fit for job",
            "context": f"Job: {job_description}\n\nResume: {candidate['resume']}",
            "expected_output": "Score (0-100) and recommendation",
            "external_user_id": external_user_id
        },
        auth=AUTH
    )
    if response.status_code == 202:
        return {
            "candidate": candidate["name"],
            "task_id": response.json()["task_id"]
        }
    else:
        raise Exception(f"Submit failed: {response.status_code}")

with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    task_ids = list(executor.map(submit_candidate, candidates))

print(f"✓ Submitted {len(task_ids)} tasks\n")

# Step 2: Poll all tasks
print("Polling results...")
results = []

def poll_candidate(task_info):
    max_attempts = 60
    for attempt in range(max_attempts):
        response = requests.get(
            f"{API_BASE}/api/v1/workers/recruitment_v1/tasks/{task_info['task_id']}",
            auth=AUTH
        )
        task = response.json()

        if task["status"] in ["done", "failed"]:
            return {
                "candidate": task_info["candidate"],
                "status": task["status"],
                "result": task.get("result")
            }

        time.sleep(2)

    return {
        "candidate": task_info["candidate"],
        "status": "timeout",
        "result": None
    }

with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(poll_candidate, task_ids))

print(f"✓ Received {len(results)} results\n")

# Step 3: Aggregate & rank
print("Ranking candidates...")
scores = []

for result in results:
    if result["status"] == "done":
        r = result["result"]
        scores.append({
            "candidate": result["candidate"],
            "score": r.get("overall_score", 0),
            "decision": r.get("decision"),
            "reasoning": r.get("reasoning")
        })
    else:
        scores.append({
            "candidate": result["candidate"],
            "score": 0,
            "decision": "REJECT",
            "reasoning": f"Error: {result['status']}"
        })

# Sort by score descending
scores.sort(key=lambda x: x["score"], reverse=True)

# Step 4: Display top 10
print("\nTOP 10 CANDIDATES:")
print("=" * 60)
for i, cand in enumerate(scores[:10]):
    print(f"{i+1}. {cand['candidate']:20s} | Score: {cand['score']:3d}/100 | {cand['decision']:6s}")

# Summary
passed = sum(1 for s in scores if s["decision"] == "PASS")
maybe = sum(1 for s in scores if s["decision"] == "MAYBE")
rejected = sum(1 for s in scores if s["decision"] == "REJECT")

print("=" * 60)
print(f"Summary: {passed} PASS, {maybe} MAYBE, {rejected} REJECT")
```

### Timing & Expectations

- **Submit:** 50 requests × 20 req/sec = ~2.5 seconds
- **Execution:** Each task 2-5 minutes (depends on worker load)
- **Polling:** 10 threads, 2-sec interval = ~2-5 minutes total
- **Total:** ~5-10 minutes end-to-end

---

## Workflow 2: Parallel Worker Execution

**Scenario:** Run multiple workers on same input, correlate results.

### Overview

```
Recruitment Input
  ↓
[Submit to recruitment_v1] → Candidate screening (parallel)
[Submit to marketing_strategy] → Market analysis
  ↓
Poll both workers
  ↓
Correlate results → unified recommendation
```

### Implementation (Python)

```python
import asyncio
import requests

async def run_parallel_workers(
    recruitment_context,
    marketing_context,
    external_user_id
):
    """Execute multiple workers in parallel"""

    # Submit to both workers
    def submit_task(worker_id, title, context):
        response = requests.post(
            f"{API_BASE}/api/v1/workers/{worker_id}/tasks",
            json={
                "title": title,
                "objective": f"Analyze: {title}",
                "context": context,
                "external_user_id": external_user_id
            },
            auth=AUTH
        )
        return response.json()["task_id"]

    # Submit in parallel
    recruitment_task_id = submit_task(
        "recruitment_v1",
        "Candidate Screening",
        recruitment_context
    )
    marketing_task_id = submit_task(
        "marketing_strategy",
        "Market Strategy",
        marketing_context
    )

    print(f"Recruitment task: {recruitment_task_id}")
    print(f"Marketing task: {marketing_task_id}\n")

    # Poll both in parallel
    async def poll_async(worker_id, task_id):
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            None,
            lambda: poll_task(worker_id, task_id)
        )

    def poll_task(worker_id, task_id):
        # Blocking poll function
        for _ in range(60):
            response = requests.get(
                f"{API_BASE}/api/v1/workers/{worker_id}/tasks/{task_id}",
                auth=AUTH
            )
            task = response.json()
            if task["status"] in ["done", "failed"]:
                return task
            time.sleep(2)
        return None

    # Poll both in parallel
    recruitment_result, marketing_result = await asyncio.gather(
        poll_async("recruitment_v1", recruitment_task_id),
        poll_async("marketing_strategy", marketing_task_id)
    )

    # Correlate results
    return {
        "recruitment": recruitment_result["result"],
        "marketing": marketing_result["result"],
        "timestamp": time.time()
    }

# Usage
results = asyncio.run(
    run_parallel_workers(
        recruitment_context="Candidate: Alice Johnson, 7 years Python",
        marketing_context="Target market: US tech companies, budget: $50k",
        external_user_id="team-1"
    )
)

print("Combined Results:")
print(f"Recruitment: {results['recruitment']}")
print(f"Marketing: {results['marketing']}")
```

---

## Workflow 3: Failure Handling & Retry

**Scenario:** Handle task failures gracefully, implement retry logic.

### Implementation

```python
def evaluate_with_retry(
    candidate_name,
    external_user_id,
    max_retries=3
):
    """Evaluate with retry on failure"""

    for attempt in range(1, max_retries + 1):
        print(f"Attempt {attempt}/{max_retries}: {candidate_name}")

        try:
            # Submit
            task = requests.post(
                f"{API_BASE}/api/v1/workers/recruitment_v1/tasks",
                json={
                    "title": f"Screen {candidate_name}",
                    "objective": "Evaluate",
                    "external_user_id": external_user_id
                },
                auth=AUTH
            ).json()

            task_id = task["task_id"]

            # Poll
            for _ in range(60):
                task = requests.get(
                    f"{API_BASE}/api/v1/workers/recruitment_v1/tasks/{task_id}",
                    auth=AUTH
                ).json()

                if task["status"] == "done":
                    return task["result"]
                elif task["status"] == "failed":
                    print(f"  Task failed: {task['result']['error']}")
                    break  # Retry

                time.sleep(2)

        except Exception as e:
            print(f"  Error: {e}")

        if attempt < max_retries:
            print(f"  Retrying in 5 seconds...")
            time.sleep(5)

    raise Exception(f"Failed after {max_retries} attempts")

# Usage
try:
    result = evaluate_with_retry("Alice Johnson", "user-1")
    print(f"✓ Success: {result}")
except Exception as e:
    print(f"✗ Failed: {e}")
```

---

## Workflow 4: Task Monitoring Dashboard

**Scenario:** Track all submitted tasks for a user in real-time.

### Implementation

```python
def monitor_user_tasks(external_user_id, worker_id="recruitment_v1"):
    """Display user's task status"""

    # List all tasks
    response = requests.get(
        f"{API_BASE}/api/v1/workers/{worker_id}/tasks",
        params={
            "external_user_id": external_user_id,
            "limit": 100
        },
        auth=AUTH
    )

    tasks = response.json()["items"]

    # Count by status
    statuses = {}
    for task in tasks:
        status = task["status"]
        statuses[status] = statuses.get(status, 0) + 1

    # Display
    print(f"User: {external_user_id}")
    print(f"Total tasks: {len(tasks)}")
    print("\nBreakdown by status:")
    for status, count in statuses.items():
        print(f"  {status:10s}: {count:3d}")

    # Show recent tasks
    print("\nRecent tasks:")
    for task in tasks[:5]:
        print(f"  {task['task_id']}: {task['status']}")
```

---

## Best Practices

### 1. Use External User IDs
Always include `external_user_id` for tracking and filtering:
```python
external_user_id = f"ceonova-user-{user_id}"
```

### 2. Implement Exponential Backoff
```python
delay = 2
max_delay = 30
for _ in range(max_retries):
    # ... poll ...
    delay = min(delay * 2, max_delay)
    time.sleep(delay)
```

### 3. Handle Rate Limiting
```python
if response.status_code == 429:
    print("Rate limited, waiting 1 minute...")
    time.sleep(60)
```

### 4. Monitor Failure Rates
```python
failures = sum(1 for r in results if r["status"] != "done")
failure_rate = failures / len(results)
if failure_rate > 0.2:  # > 20%
    print(f"⚠️  High failure rate: {failure_rate*100:.1f}%")
```

### 5. Use Logging
```python
import logging
logger = logging.getLogger(__name__)
logger.info(f"Submitted task: {task_id}")
logger.error(f"Task failed: {error}")
```

---

## Next Steps

→ **[Async Patterns](03-async-patterns.md)** — Webhooks, polling, idempotency

→ **[Troubleshooting](../operations/01-troubleshooting.md)** — Debug failures

→ **[Code Examples](code-examples/01-curl.md)** — Copy-paste ready samples
