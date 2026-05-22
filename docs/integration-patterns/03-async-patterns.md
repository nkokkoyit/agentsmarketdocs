# Async Patterns & Webhooks

**Implement robust async communication with AgentsMarket**

---

## Current (v1): Polling

AgentsMarket v1 uses **polling** for task status updates. Webhooks are planned for Q3 2026.

### Polling Overview

Client periodically calls `/api/v1/workers/{worker_id}/tasks/{task_id}` to check status.

**Pros:**
- Simple to implement
- No callback endpoint required
- Easy to debug

**Cons:**
- Latency: Min 2-5 seconds (poll interval)
- Network cost: Steady polling drain
- Throughput: Max ~1 task/2 sec per connection

### Polling Best Practices

#### 1. Exponential Backoff

Start with short delay, increase gradually:

```python
def poll_with_backoff(worker_id, task_id, max_retries=30):
    delay = 2  # Start 2 seconds
    
    for attempt in range(max_retries):
        response = requests.get(
            f"http://localhost:8000/api/v1/workers/{worker_id}/tasks/{task_id}",
            auth=AUTH
        )
        task = response.json()
        
        if task["status"] in ["done", "failed"]:
            return task
        
        print(f"Waiting {delay}s before retry...")
        time.sleep(delay)
        delay = min(delay * 2, 30)  # Double each time, cap at 30s
    
    raise TimeoutError("Polling timeout")
```

**Benefits:**
- Reduces server load (fewer early polls)
- Still responsive (short initial delay)
- Handles slow workers gracefully (long max delay)

#### 2. Circuit Breaker

Stop polling if task is stuck:

```python
def poll_with_circuit_breaker(worker_id, task_id):
    max_age_minutes = 10
    
    # Get task
    response = requests.get(
        f"http://localhost:8000/api/v1/workers/{worker_id}/tasks/{task_id}",
        auth=AUTH
    )
    task = response.json()
    
    # Check age
    created = datetime.fromisoformat(task["created_at"].replace("Z", "+00:00"))
    age = datetime.now(timezone.utc) - created
    
    if age.total_seconds() > max_age_minutes * 60:
        raise Exception(f"Task exceeded max age ({max_age_minutes} min)")
    
    return task
```

#### 3. Idempotent Processing

Handle duplicate results safely:

```python
def process_result_idempotent(task_id, result):
    """Process result only once"""
    
    # Check if already processed
    if is_processed(task_id):
        print(f"Task {task_id} already processed, skipping")
        return
    
    # Process
    handle_result(result)
    
    # Mark as processed
    mark_processed(task_id)

def is_processed(task_id):
    """Check processing status in DB"""
    return db.processed_tasks.exists(task_id)

def mark_processed(task_id):
    """Record that we processed this task"""
    db.processed_tasks.set(task_id, datetime.now())
```

---

## Future (Q3 2026): Webhooks

### Webhook Contract (Preview)

When task status changes, AgentsMarket will POST to your webhook endpoint:

```json
{
  "event": "task_status_changed",
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "done",
  "result": {
    "overall_score": 85,
    "decision": "PASS"
  },
  "timestamp": "2026-05-23T10:35:00Z",
  "attempt": 1
}
```

### Signature Verification (Preview)

Header: `X-AgentsMarket-Signature: sha256=<hmac_hash>`

```python
import hmac
import hashlib

def verify_webhook_signature(request, webhook_secret):
    """Verify webhook came from AgentsMarket"""
    
    signature = request.headers.get("X-AgentsMarket-Signature", "")
    body = request.get_data()
    
    # Compute expected signature
    expected = "sha256=" + hmac.new(
        webhook_secret.encode(),
        body,
        hashlib.sha256
    ).hexdigest()
    
    # Compare (constant-time)
    if not hmac.compare_digest(signature, expected):
        raise ValueError("Invalid signature")
    
    return True
```

### Webhook Receiver (Flask Example)

```python
from flask import Flask, request
import json
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret-from-settings"

@app.route("/webhooks/agentsmarket", methods=["POST"])
def handle_webhook():
    """Receive task status updates"""
    
    # 1. Verify signature
    try:
        verify_webhook_signature(request, WEBHOOK_SECRET)
    except ValueError:
        return {"error": "Invalid signature"}, 401
    
    # 2. Parse payload
    payload = request.get_json()
    task_id = payload["task_id"]
    status = payload["status"]
    result = payload.get("result")
    attempt = payload.get("attempt", 1)
    
    # 3. Idempotency check (handle retries)
    if is_webhook_processed(task_id, attempt):
        print(f"Duplicate webhook (task {task_id}, attempt {attempt}), ignoring")
        return {"status": "ok"}, 200
    
    # 4. Process result
    if status == "done":
        handle_task_done(task_id, result)
    elif status == "failed":
        handle_task_failed(task_id, result)
    
    # 5. Acknowledge receipt
    mark_webhook_processed(task_id, attempt)
    return {"status": "ok"}, 200

def is_webhook_processed(task_id, attempt):
    """Check if we've already processed this webhook"""
    key = f"webhook:{task_id}:{attempt}"
    return cache.exists(key)

def mark_webhook_processed(task_id, attempt):
    """Record that we've processed this webhook"""
    key = f"webhook:{task_id}:{attempt}"
    cache.set(key, True, ttl=3600)  # Expire after 1 hour

def handle_task_done(task_id, result):
    print(f"✅ Task {task_id} done")
    print(f"Score: {result['overall_score']}, Decision: {result['decision']}")
    # Update DB, trigger notifications, etc.

def handle_task_failed(task_id, result):
    print(f"❌ Task {task_id} failed")
    print(f"Error: {result['error']}")
    # Log error, retry, escalate, etc.
```

### Webhook vs. Polling

| Aspect | Polling | Webhooks |
|--------|---------|----------|
| **Latency** | 2-5s (poll interval) | <100ms (push) |
| **Throughput** | Limited (rate limit) | Many parallel tasks |
| **Setup** | Simple | Requires endpoint |
| **Complexity** | Poll loop | Signature verification, dedup, receiver |
| **Network** | Steady polling | Only on state change |
| **Failure recovery** | Client-side retry | AgentsMarket retries |

---

## Async/Await Pattern (JavaScript)

```javascript
async function waitForTask(workerId, taskId) {
  let delay = 2000;  // 2 seconds

  for (let attempt = 1; attempt <= 30; attempt++) {
    const task = await fetch(
      `http://localhost:8000/api/v1/workers/${workerId}/tasks/${taskId}`,
      { headers: authHeaders }
    ).then(r => r.json());

    console.log(`[${attempt}] Status: ${task.status}`);

    if (["done", "failed"].includes(task.status)) {
      return task;
    }

    await new Promise(resolve => setTimeout(resolve, delay));
    delay = Math.min(delay * 2, 30000);  // Exponential backoff
  }

  throw new Error("Polling timeout");
}

// Usage
const task = await waitForTask("recruitment_v1", taskId);
console.log(`Result:`, task.result);
```

---

## Dead-Letter Queue Pattern

Handle failed tasks separately:

```python
from collections import deque
from datetime import datetime

class TaskQueue:
    def __init__(self):
        self.pending = deque()
        self.dead_letter = deque()
    
    def submit(self, task_data):
        task_id = submit_task(**task_data)
        self.pending.append({
            "task_id": task_id,
            "data": task_data,
            "submitted_at": datetime.now(),
            "attempt": 1
        })
        return task_id
    
    def process_results(self):
        """Poll pending tasks, move failures to DLQ"""
        
        while self.pending:
            task_item = self.pending[0]
            task_id = task_item["task_id"]
            
            try:
                task = poll_task_once(worker_id, task_id)
                
                if task["status"] == "done":
                    self.pending.popleft()
                    return task["result"]
                
                elif task["status"] == "failed":
                    # Move to dead-letter queue
                    self.pending.popleft()
                    task_item["error"] = task["result"]["error"]
                    self.dead_letter.append(task_item)
                    print(f"❌ Task {task_id} failed, moved to DLQ")
            
            except Exception as e:
                print(f"⚠️  Poll error: {e}")
            
            time.sleep(2)
    
    def retry_dead_letter(self):
        """Retry failed tasks"""
        
        while self.dead_letter:
            task_item = self.dead_letter[0]
            
            # Check retry limit
            if task_item["attempt"] >= 3:
                print(f"⚠️  Task {task_item['task_id']} exceeded max retries")
                self.dead_letter.popleft()
                continue
            
            # Retry
            print(f"🔄 Retrying task (attempt {task_item['attempt']+1}/3)")
            task_item["attempt"] += 1
            task_id = submit_task(**task_item["data"])
            task_item["task_id"] = task_id
            
            # Move back to pending
            self.pending.append(self.dead_letter.popleft())
```

---

## Monitoring & Observability

### Request/Response Logging

```python
import logging
import json

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

def log_request(method, url, **kwargs):
    logger.debug(f"{method} {url}")
    if "json" in kwargs:
        logger.debug(f"  Body: {json.dumps(kwargs['json'], indent=2)}")

def log_response(response):
    logger.debug(f"  Status: {response.status_code}")
    try:
        logger.debug(f"  Body: {json.dumps(response.json(), indent=2)}")
    except:
        logger.debug(f"  Body: {response.text[:500]}")
```

### Metrics

```python
from time import time

class TaskMetrics:
    def __init__(self):
        self.submitted = 0
        self.completed = 0
        self.failed = 0
        self.total_time = 0
    
    def record_submission(self):
        self.submitted += 1
    
    def record_completion(self, elapsed_seconds):
        self.completed += 1
        self.total_time += elapsed_seconds
    
    def record_failure(self):
        self.failed += 1
    
    def summary(self):
        avg_time = self.total_time / self.completed if self.completed > 0 else 0
        return {
            "submitted": self.submitted,
            "completed": self.completed,
            "failed": self.failed,
            "average_time_seconds": avg_time,
            "success_rate": f"{self.completed/self.submitted*100:.1f}%" if self.submitted > 0 else "N/A"
        }
```

---

## Next Steps

→ **[Troubleshooting](../operations/01-troubleshooting.md)** — Debug common issues

→ **[Code Examples](code-examples/01-curl.md)** — Working samples

→ **[Operations](../operations/01-troubleshooting.md)** — Monitoring & alerts
