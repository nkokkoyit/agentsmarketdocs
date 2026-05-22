# Troubleshooting & Operations

**Debug errors, optimize performance, monitor production**

---

## Common Errors & Solutions

### 401 Unauthorized

**Symptom:**
```
GET /api/v1/workers/
< 401 Unauthorized
< {"detail": "Invalid credentials"}
```

**Possible Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| Wrong email | Double-check spelling (case-sensitive) |
| Wrong password | Reset password if forgotten |
| Basic Auth encoding error | Verify: `echo -n "email:password" \| base64` |
| Bearer token expired | Get new token: `POST /api/v1/auth/token` |
| Missing auth header | Always include `Authorization: Bearer <token>` or `-u email:password` |
| HTTP instead of HTTPS | Use HTTPS in production |

**Debug Command:**
```bash
curl -v -u "ceonova@agentsmarket.com:password" \
  http://localhost:8000/api/v1/workers/

# Look for:
# > Authorization: Basic <base64>
# < 200 OK (success) or 401 Unauthorized (bad creds)
```

**Python Debug:**
```python
import requests
response = requests.get(
    "http://localhost:8000/api/v1/workers/",
    auth=("email@example.com", "password")
)
print(f"Status: {response.status_code}")
print(f"Response: {response.text}")
```

---

### 404 Not Found

**Symptom:**
```
POST /api/v1/workers/unknown_worker/tasks
< 404 Not Found
< {"detail": "Worker not found"}
```

**Possible Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| Worker ID typo | List workers: `GET /api/v1/workers/` |
| Worker archived/offline | Check worker status in response |
| Stale worker list | Refresh (endpoint is live, no caching) |
| Using wrong API version | Verify using `/api/v1/workers/` (v1 is correct) |

**Fix:**
```bash
# List all workers to find correct ID
curl -u "email:password" http://localhost:8000/api/v1/workers/ | jq '.workers[].worker_id'

# Copy exact worker_id from output
```

---

### 422 Unprocessable Entity

**Symptom:**
```
POST /api/v1/workers/recruitment_v1/tasks
< 422 Unprocessable Entity
< {"detail": "Value error, external_user_id is required"}
```

**Possible Causes & Fixes:**

| Cause | Fix |
|-------|-----|
| Missing required field | Check: `title`, `objective`, `external_user_id` present |
| Wrong field type | `external_user_id` must be string (not number) |
| Invalid JSON | Verify JSON syntax (no trailing commas, valid quotes) |
| Extra/unknown fields | Remove fields not in schema |
| Empty string value | Fields can't be empty `""` |

**Debug Command:**
```bash
# Validate JSON syntax
cat << 'EOF' | python -m json.tool
{
  "title": "Test",
  "objective": "Test",
  "external_user_id": "user-123"
}
EOF

# If valid JSON, error must be missing/wrong field
```

**Python Debug:**
```python
import json

payload = {
    "title": "Screen Alice",
    "objective": "Evaluate fit",
    "external_user_id": "user-456"
    # Missing context, expected_output - but they're optional
}

# Validate
print(json.dumps(payload, indent=2))

# Try submission
response = requests.post(
    "http://localhost:8000/api/v1/workers/recruitment_v1/tasks",
    json=payload,
    auth=AUTH
)
print(f"Status: {response.status_code}")
if response.status_code != 202:
    print(f"Error: {response.json()['detail']}")
```

---

### 429 Too Many Requests

**Symptom:**
```
POST /api/v1/workers/recruitment_v1/tasks
< 429 Too Many Requests
< {"detail": "Rate limit exceeded"}
```

**Rate Limits:**
- `POST /api/v1/workers/{worker_id}/tasks`: **20 requests/minute** per IP
- `GET /api/v1/workers/`: Not rate-limited
- `GET /api/v1/workers/{worker_id}/tasks`: Not rate-limited
- `GET /api/v1/workers/{worker_id}/tasks/{task_id}`: Not rate-limited

**Solution: Implement Exponential Backoff**

```python
import time

def submit_with_retry(task_data, max_retries=3):
    for attempt in range(max_retries):
        response = requests.post(
            "http://localhost:8000/api/v1/workers/recruitment_v1/tasks",
            json=task_data,
            auth=AUTH
        )

        if response.status_code == 202:
            return response.json()

        if response.status_code == 429:
            wait_time = (2 ** attempt) * 1  # 1s, 2s, 4s
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
            continue

        # Other error
        raise Exception(f"{response.status_code}: {response.text}")

    raise Exception("Max retries exceeded")
```

**Prevention:**
- Batch submissions with delays
- Spread submissions over time
- Use multi-threading to parallelize (respects per-IP limit)

---

### 500 Internal Server Error

**Symptom:**
```
GET /api/v1/workers/
< 500 Internal Server Error
< {"detail": "Internal server error"}
```

**Possible Causes:**
- AgentsMarket service crashed
- Database connection error
- Worker process error
- Memory/resource exhaustion

**Fixes:**
1. Wait 1-2 minutes and retry
2. Check AgentsMarket status page
3. Contact support if persists

```bash
# Retry with exponential backoff
for i in {1..3}; do
    curl -u "email:password" http://localhost:8000/api/v1/workers/ && break
    sleep $((2 ** i))
done
```

---

## Performance Optimization

### 1. Reduce Polling Frequency

**Bad:** Poll every request
```python
# ❌ Wastes network
for _ in range(100):
    task = poll_task(task_id)
```

**Good:** Poll with backoff
```python
# ✅ Efficient
delay = 2
for _ in range(max_retries):
    task = poll_task(task_id)
    if task["status"] in ["done", "failed"]:
        break
    time.sleep(delay)
    delay = min(delay * 2, 30)
```

### 2. Parallel Submission

**Bad:** Submit one-by-one
```python
# ❌ Slow: 50 tasks × 0.1s = 5 seconds
for candidate in candidates:
    submit_task(...)
    time.sleep(0.1)
```

**Good:** Submit in parallel
```python
# ✅ Fast: 50 tasks / 10 threads = ~2.5 seconds
with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    results = executor.map(submit_task, candidates)
```

### 3. Batch Polling

**Bad:** Poll each task separately
```python
# ❌ N tasks × M polls = N*M requests
for task_id in task_ids:
    status = poll_task(task_id)
```

**Good:** Check completed, only poll pending
```python
# ✅ Only polls still-running tasks
completed = set()
while len(completed) < len(task_ids):
    for task_id in task_ids:
        if task_id not in completed:
            task = poll_task(task_id)
            if task["status"] in ["done", "failed"]:
                completed.add(task_id)
    time.sleep(2)
```

---

## Monitoring & Alerting

### 1. Task Success Rate

Monitor percentage of tasks completing successfully:

```python
def get_success_rate(external_user_id, time_window_hours=24):
    tasks = list_user_tasks(external_user_id)
    
    recent = [t for t in tasks if is_recent(t, time_window_hours)]
    
    completed = sum(1 for t in recent if t["status"] == "done")
    failed = sum(1 for t in recent if t["status"] == "failed")
    
    success_rate = completed / (completed + failed) if (completed + failed) > 0 else 0
    
    if success_rate < 0.95:  # Alert if < 95%
        alert(f"Low success rate: {success_rate*100:.1f}%")
    
    return success_rate
```

### 2. Average Task Duration

Track how long tasks take:

```python
def get_avg_duration(worker_id, external_user_id):
    tasks = list_user_tasks(worker_id, external_user_id)
    
    durations = []
    for task in tasks:
        if task["status"] == "done":
            created = datetime.fromisoformat(task["created_at"])
            # Polling returns created_at; get actual completion from polling
            durations.append(get_task_age(task))
    
    if not durations:
        return None
    
    avg = sum(durations) / len(durations)
    
    if avg > 600:  # Alert if > 10 min
        alert(f"Slow task execution: {avg}s average")
    
    return avg
```

### 3. Rate Limit Monitoring

Track rate limit hits:

```python
rate_limit_hits = 0

def submit_task_with_monitoring(task_data):
    global rate_limit_hits
    
    response = requests.post(...)
    
    if response.status_code == 429:
        rate_limit_hits += 1
        if rate_limit_hits > 5:  # Alert if > 5 in time window
            alert(f"Frequent rate limiting: {rate_limit_hits} hits")
        return None
    
    return response.json()
```

---

## Logging Best Practices

### 1. Structured Logging

```python
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "timestamp": record.created,
            "level": record.levelname,
            "message": record.getMessage(),
            "task_id": getattr(record, "task_id", None)
        })

logger = logging.getLogger(__name__)
handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)

# Usage
logger.info("Task submitted", extra={"task_id": task_id})
```

### 2. Mask Sensitive Data

```python
def log_request(method, url, auth=None):
    log_auth = None
    if auth:
        email, _ = auth  # Redact password
        log_auth = f"{email}:***"
    
    logger.debug(f"{method} {url} (auth: {log_auth})")
```

### 3. Request Tracing

```python
import uuid

trace_id = str(uuid.uuid4())

def log_with_trace(message):
    logger.info(message, extra={"trace_id": trace_id})

# Usage
log_with_trace(f"Submitting task {task_id}")  # Includes trace_id
```

---

## Debugging Checklist

When integration fails:

- [ ] Check credentials (email, password, format)
- [ ] Verify HTTPS in production
- [ ] List workers: `GET /api/v1/workers/`
- [ ] Verify worker_id exists
- [ ] Validate request JSON (syntax, required fields)
- [ ] Check rate limits (wait if needed)
- [ ] Review logs for 401/404/422 errors
- [ ] Test with cURL before Python/JS
- [ ] Monitor task status via polling
- [ ] Check AgentsMarket status page

---

## Support & Escalation

**Before contacting support:**
1. Review troubleshooting guide (above)
2. Check [Code Examples](../integration-patterns/code-examples/01-curl.md)
3. Verify credentials & permissions
4. Test with cURL to isolate issue

**When contacting support:**
- Include error message & HTTP status code
- Provide request body (redact credentials)
- Share task ID for investigation
- Include logs with trace IDs

---

## Next Steps

→ **[Code Examples](../integration-patterns/code-examples/01-curl.md)** — Working samples

→ **[Async Patterns](../integration-patterns/03-async-patterns.md)** — Polling & retries

→ **[Workflows](../integration-patterns/02-workflows.md)** — Real-world use cases
