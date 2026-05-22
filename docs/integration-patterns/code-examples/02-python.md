# Python Examples

**Ready-to-use Python samples using requests library**

---

## Setup

### Installation

```bash
pip install requests python-dotenv
```

### Configuration

Create a file `agentsmarket.py`:

```python
import os
import base64
import requests
import time
from typing import Optional, Dict, Any
from dotenv import load_dotenv

load_dotenv()

API_BASE = os.getenv("AGENTSMARKET_API", "http://localhost:8000")
AUTH_EMAIL = os.getenv("CEONOVA_EMAIL", "ceonova@agentsmarket.com")
AUTH_PASSWORD = os.getenv("CEONOVA_PASSWORD", "your-password")

AUTH = (AUTH_EMAIL, AUTH_PASSWORD)
```

Create `.env` file:

```env
AGENTSMARKET_API=http://localhost:8000
CEONOVA_EMAIL=ceonova@agentsmarket.com
CEONOVA_PASSWORD=your-password
```

---

## Example 1: Discover Workers

```python
def discover_workers() -> Dict[str, Any]:
    """List all available workers"""
    response = requests.get(
        f"{API_BASE}/api/v1/workers/",
        auth=AUTH
    )
    response.raise_for_status()  # Raise exception if status != 200
    return response.json()

# Usage
workers = discover_workers()
for worker in workers["workers"]:
    print(f"✓ {worker['worker_id']}: {worker['name']}")
    print(f"  Capabilities: {', '.join(worker['capabilities'])}")
```

**Output:**
```
✓ recruitment_v1: Recruitment Screening
  Capabilities: screening, skill_matching, scoring, recommendation
✓ marketing_strategy: Marketing Strategy
  Capabilities: market_research, competitive_analysis, campaign_planning, content_creation
```

---

## Example 2: Get Worker Details

```python
def get_worker_details(worker_id: str) -> Dict[str, Any]:
    """Retrieve detailed info about a worker"""
    response = requests.get(
        f"{API_BASE}/api/v1/workers/{worker_id}",
        auth=AUTH
    )
    response.raise_for_status()
    return response.json()

# Usage
worker = get_worker_details("recruitment_v1")
print(f"Name: {worker['name']}")
print(f"Description: {worker['description']}")
print(f"Capabilities: {worker['capabilities']}")
```

---

## Example 3: Submit a Task

```python
def submit_task(
    worker_id: str,
    title: str,
    objective: str,
    external_user_id: str,
    context: Optional[str] = None,
    expected_output: Optional[str] = None
) -> Dict[str, str]:
    """Submit a task to a worker"""
    
    payload = {
        "title": title,
        "objective": objective,
        "external_user_id": external_user_id,
    }
    
    if context:
        payload["context"] = context
    if expected_output:
        payload["expected_output"] = expected_output

    response = requests.post(
        f"{API_BASE}/api/v1/workers/{worker_id}/tasks",
        json=payload,
        auth=AUTH
    )
    
    if response.status_code == 202:
        return response.json()  # {"task_id": "...", "worker_id": "..."}
    else:
        raise Exception(f"Submit failed ({response.status_code}): {response.text}")

# Usage
task = submit_task(
    worker_id="recruitment_v1",
    title="Screen Alice Johnson for Senior Engineer",
    objective="Evaluate candidate fit for role",
    external_user_id="ceonova-user-456",
    context="Job: Senior Backend Engineer, 5+ years Python, Docker, AWS",
    expected_output="Score (0-100) and recommendation (PASS/MAYBE/REJECT)"
)

print(f"✓ Task submitted!")
print(f"  Task ID: {task['task_id']}")
print(f"  Worker: {task['worker_id']}")
```

---

## Example 4: List User's Tasks

```python
def list_user_tasks(
    worker_id: str,
    external_user_id: str,
    limit: int = 20,
    offset: int = 0
) -> Dict[str, Any]:
    """List all tasks submitted by a CEONova user"""
    
    params = {
        "external_user_id": external_user_id,
        "limit": limit,
        "offset": offset
    }
    
    response = requests.get(
        f"{API_BASE}/api/v1/workers/{worker_id}/tasks",
        params=params,
        auth=AUTH
    )
    response.raise_for_status()
    return response.json()

# Usage
tasks = list_user_tasks(
    worker_id="recruitment_v1",
    external_user_id="ceonova-user-456",
    limit=10
)

print(f"Total tasks: {tasks['total']}")
for task in tasks["items"]:
    print(f"  • {task['task_id']}: {task['status']}")
```

---

## Example 5: Poll Task Status (Single Poll)

```python
def poll_task_once(
    worker_id: str,
    task_id: str,
    external_user_id: Optional[str] = None
) -> Dict[str, Any]:
    """Check task status once"""
    
    params = {}
    if external_user_id:
        params["external_user_id"] = external_user_id
    
    response = requests.get(
        f"{API_BASE}/api/v1/workers/{worker_id}/tasks/{task_id}",
        params=params,
        auth=AUTH
    )
    response.raise_for_status()
    return response.json()

# Usage
task = poll_task_once("recruitment_v1", "550e8400-e29b-41d4-a716-446655440000")
print(f"Status: {task['status']}")
if task['status'] == 'done':
    print(f"Result: {task['result']}")
```

---

## Example 6: Poll with Exponential Backoff

```python
def poll_task_status(
    worker_id: str,
    task_id: str,
    max_retries: int = 30,
    initial_delay: float = 2.0
) -> Dict[str, Any]:
    """Poll task status with exponential backoff"""
    
    delay = initial_delay
    
    for attempt in range(1, max_retries + 1):
        print(f"[Attempt {attempt}/{max_retries}] Polling task status...")
        
        try:
            task = poll_task_once(worker_id, task_id)
            status = task["status"]
            
            print(f"  Status: {status}")
            
            # Task complete
            if status in ["done", "failed"]:
                return task
            
            # Still running, wait and retry
            if attempt < max_retries:
                print(f"  Waiting {delay}s before retry...")
                time.sleep(delay)
                delay = min(delay * 2, 30)  # Exponential backoff, cap at 30s
        
        except requests.exceptions.RequestException as e:
            print(f"  Error: {e}")
            if attempt == max_retries:
                raise
            time.sleep(delay)
    
    raise TimeoutError(f"Task polling timeout after {max_retries} attempts")

# Usage
task = poll_task_status("recruitment_v1", "550e8400-e29b-41d4-a716-446655440000")
print(f"\n✓ Task complete!")
print(f"  Final status: {task['status']}")
if task['status'] == 'done':
    print(f"  Result: {task['result']}")
```

---

## Example 7: End-to-End Workflow

```python
def evaluate_candidate(
    candidate_name: str,
    candidate_info: str,
    job_description: str,
    external_user_id: str
) -> Dict[str, Any]:
    """Submit task and wait for result (blocking)"""
    
    print(f"Evaluating {candidate_name}...")
    print("=" * 50)
    
    # Step 1: Submit task
    print(f"\n1. Submitting task...")
    task = submit_task(
        worker_id="recruitment_v1",
        title=f"Screen {candidate_name} for recruitment",
        objective="Evaluate candidate fit for the role",
        external_user_id=external_user_id,
        context=f"Job: {job_description}\n\nCandidate: {candidate_info}",
        expected_output="Score (0-100) and recommendation (PASS/MAYBE/REJECT)"
    )
    
    print(f"   ✓ Task ID: {task['task_id']}")
    
    # Step 2: Poll for result
    print(f"\n2. Waiting for result...")
    final_task = poll_task_status("recruitment_v1", task["task_id"])
    
    # Step 3: Process result
    print(f"\n3. Result:")
    print(f"   Status: {final_task['status']}")
    
    if final_task['status'] == 'done':
        result = final_task['result']
        print(f"   Score: {result.get('overall_score')}/100")
        print(f"   Decision: {result.get('decision')}")
        print(f"   Reasoning: {result.get('reasoning')}")
    elif final_task['status'] == 'failed':
        print(f"   Error: {final_task['result'].get('error')}")
    
    print("=" * 50)
    return final_task

# Usage
result = evaluate_candidate(
    candidate_name="Alice Johnson",
    candidate_info="7 years Python experience, 3 years Docker, 2 years Kubernetes",
    job_description="Senior Backend Engineer - 5+ years Python, Docker, AWS required",
    external_user_id="ceonova-user-456"
)
```

---

## Example 8: Batch Evaluation (Parallel)

```python
import concurrent.futures

def evaluate_candidates_parallel(
    candidates: list,
    job_description: str,
    external_user_id: str,
    max_workers: int = 5
) -> list:
    """Evaluate multiple candidates in parallel"""
    
    print(f"Evaluating {len(candidates)} candidates in parallel...")
    print(f"Max concurrent tasks: {max_workers}")
    print("=" * 50)
    
    results = []
    
    def evaluate_one(candidate):
        """Evaluate single candidate"""
        name = candidate["name"]
        info = candidate["info"]
        
        print(f"\n→ {name}: Submitting...")
        task = submit_task(
            worker_id="recruitment_v1",
            title=f"Screen {name}",
            objective="Evaluate candidate fit",
            external_user_id=external_user_id,
            context=f"Job: {job_description}\n\nCandidate: {info}",
            expected_output="Score and recommendation"
        )
        return {"candidate": name, "task_id": task["task_id"]}
    
    # Submit all tasks in parallel
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        submit_results = list(executor.map(evaluate_one, candidates))
    
    print(f"\n✓ All {len(submit_results)} tasks submitted!")
    print("=" * 50)
    print("\nPolling for results...")
    print("-" * 50)
    
    # Poll all tasks in parallel
    def poll_one(submit_result):
        """Poll single task"""
        candidate = submit_result["candidate"]
        task_id = submit_result["task_id"]
        
        try:
            task = poll_task_status("recruitment_v1", task_id, max_retries=60)
            return {
                "candidate": candidate,
                "task_id": task_id,
                "status": task["status"],
                "result": task.get("result")
            }
        except Exception as e:
            return {
                "candidate": candidate,
                "task_id": task_id,
                "status": "error",
                "error": str(e)
            }
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        results = list(executor.map(poll_one, submit_results))
    
    # Display results
    print("\n" + "=" * 50)
    print("RESULTS:")
    print("=" * 50)
    
    for result in results:
        status = result["status"]
        candidate = result["candidate"]
        
        if status == "done":
            r = result["result"]
            score = r.get("overall_score", "?")
            decision = r.get("decision", "?")
            print(f"✓ {candidate}: {score}/100 - {decision}")
        elif status == "error":
            print(f"✗ {candidate}: ERROR - {result['error']}")
        else:
            print(f"? {candidate}: {status}")
    
    return results

# Usage
candidates = [
    {"name": "Alice Johnson", "info": "7 yrs Python, 3 yrs Docker"},
    {"name": "Bob Smith", "info": "5 yrs Python, 1 yr Kubernetes"},
    {"name": "Carol White", "info": "10 yrs backend, AWS expert"},
]

results = evaluate_candidates_parallel(
    candidates=candidates,
    job_description="Senior Backend Engineer - 5+ years Python required",
    external_user_id="ceonova-team-1",
    max_workers=3
)
```

---

## Example 9: Error Handling

```python
def safe_submit_task(
    worker_id: str,
    title: str,
    objective: str,
    external_user_id: str
) -> Dict[str, Any]:
    """Submit with proper error handling"""
    
    try:
        task = submit_task(
            worker_id=worker_id,
            title=title,
            objective=objective,
            external_user_id=external_user_id
        )
        return {"success": True, "data": task}
    
    except requests.exceptions.HTTPError as e:
        status_code = e.response.status_code
        
        if status_code == 401:
            return {
                "success": False,
                "error": "Authentication failed",
                "reason": "Check credentials in .env"
            }
        elif status_code == 404:
            return {
                "success": False,
                "error": "Worker not found",
                "reason": "List workers and verify worker_id"
            }
        elif status_code == 422:
            return {
                "success": False,
                "error": "Invalid request",
                "reason": "Check request body matches schema"
            }
        elif status_code == 429:
            return {
                "success": False,
                "error": "Rate limited",
                "reason": "Wait 1 minute before retrying"
            }
        else:
            return {
                "success": False,
                "error": f"HTTP {status_code}",
                "reason": str(e)
            }
    
    except Exception as e:
        return {
            "success": False,
            "error": "Unexpected error",
            "reason": str(e)
        }

# Usage
result = safe_submit_task(
    worker_id="recruitment_v1",
    title="Test",
    objective="Test objective",
    external_user_id="test-user"
)

if result["success"]:
    print(f"✓ Task submitted: {result['data']['task_id']}")
else:
    print(f"✗ Error: {result['error']}")
    print(f"  {result['reason']}")
```

---

## Example 10: Logging & Monitoring

```python
import logging
from datetime import datetime

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

def evaluate_with_logging(
    candidate_name: str,
    external_user_id: str
) -> Dict[str, Any]:
    """Evaluate candidate with detailed logging"""
    
    logger.info(f"Starting evaluation for {candidate_name}")
    
    try:
        # Submit
        logger.info(f"Submitting task for {candidate_name}")
        task = submit_task(
            worker_id="recruitment_v1",
            title=f"Screen {candidate_name}",
            objective="Evaluate fit",
            external_user_id=external_user_id
        )
        task_id = task["task_id"]
        logger.info(f"Task submitted: {task_id}")
        
        # Poll
        logger.info(f"Polling task {task_id}")
        start_time = datetime.now()
        final_task = poll_task_status("recruitment_v1", task_id)
        elapsed = (datetime.now() - start_time).total_seconds()
        
        logger.info(f"Task {task_id} completed in {elapsed:.1f}s")
        logger.info(f"Final status: {final_task['status']}")
        
        if final_task['status'] == 'done':
            result = final_task['result']
            logger.info(f"Score: {result.get('overall_score')}/100, Decision: {result.get('decision')}")
        
        return final_task
    
    except Exception as e:
        logger.error(f"Evaluation failed: {str(e)}", exc_info=True)
        raise

# Usage
try:
    result = evaluate_with_logging(
        candidate_name="Alice Johnson",
        external_user_id="ceonova-user-456"
    )
except Exception as e:
    logger.error(f"Fatal error: {e}")
```

---

## Tips & Best Practices

### Use Environment Variables
```python
from dotenv import load_dotenv
import os

load_dotenv()  # Load .env file
email = os.getenv("CEONOVA_EMAIL")
password = os.getenv("CEONOVA_PASSWORD")
```

### Implement Retry Logic
```python
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(total=3, backoff_factor=0.5)
adapter = HTTPAdapter(max_retries=retry)
session.mount('http://', adapter)
session.mount('https://', adapter)

response = session.get(f"{API_BASE}/api/v1/workers/", auth=AUTH)
```

### Use Context Managers for Resources
```python
with requests.Session() as session:
    response = session.get(f"{API_BASE}/api/v1/workers/", auth=AUTH)
    # Session automatically closed
```

### Validate Responses
```python
def safe_json_parse(response):
    try:
        return response.json()
    except ValueError:
        raise ValueError(f"Invalid JSON response: {response.text}")
```

---

## Next Steps

→ **[JavaScript Examples](03-javascript.md)** — Use Fetch API

→ **[Workflows](../02-workflows.md)** — Real-world patterns

→ **[Troubleshooting](../../operations/01-troubleshooting.md)** — Debug errors
