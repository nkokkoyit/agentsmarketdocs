# cURL Examples

**Ready-to-use cURL samples for AgentsMarket API**

---

## Basics

### Authentication
cURL supports HTTP Basic Auth via `-u` flag:

```bash
curl -u "email@example.com:password" http://api.example.com/endpoint
```

### Content Type
For POST requests with JSON:

```bash
curl -X POST http://api.example.com/endpoint \
  -H "Content-Type: application/json" \
  -d '{json_data}'
```

### Verbose Mode
See headers, status codes, and full request/response:

```bash
curl -v -X GET http://api.example.com/endpoint \
  -u "email:password"
```

---

## Example 1: Discover Workers

**List all available workers**

```bash
curl -X GET http://localhost:8000/api/v1/workers/ \
  -u "ceonova@agentsmarket.com:password"
```

**Expected Output:**
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

---

## Example 2: Get Worker Details

**Retrieve details about a specific worker**

```bash
curl -X GET http://localhost:8000/api/v1/workers/recruitment_v1 \
  -u "ceonova@agentsmarket.com:password"
```

**Expected Output:**
```json
{
  "worker_id": "recruitment_v1",
  "name": "Recruitment Screening",
  "description": "Evaluate candidates against job criteria",
  "status": "available",
  "capabilities": ["screening", "skill_matching", "scoring", "recommendation"]
}
```

---

## Example 3: Submit a Task

**Queue work for a worker**

```bash
curl -X POST http://localhost:8000/api/v1/workers/recruitment_v1/tasks \
  -u "ceonova@agentsmarket.com:password" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Screen Alice Johnson for Senior Engineer",
    "objective": "Evaluate candidate fit for role",
    "context": "Job: Senior Backend Engineer, 5+ years Python, Docker, AWS. Resume: 7 years Python, 3 years Docker, 2 years Kubernetes",
    "expected_output": "Score (0-100) and recommendation (PASS/MAYBE/REJECT)",
    "external_user_id": "ceonova-user-456"
  }'
```

**Expected Output (202 Accepted):**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1"
}
```

**Save task ID for polling:**
```bash
TASK_ID="550e8400-e29b-41d4-a716-446655440000"
```

---

## Example 4: List User's Tasks

**Get all tasks submitted by a specific CEONova user**

```bash
curl -X GET "http://localhost:8000/api/v1/workers/recruitment_v1/tasks?external_user_id=ceonova-user-456&limit=20" \
  -u "ceonova@agentsmarket.com:password"
```

**Expected Output:**
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

---

## Example 5: Poll Task Status (Single Request)

**Check task progress**

```bash
curl -X GET "http://localhost:8000/api/v1/workers/recruitment_v1/tasks/550e8400-e29b-41d4-a716-446655440000" \
  -u "ceonova@agentsmarket.com:password"
```

**If running:**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "running",
  "result": null
}
```

**If done:**
```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "worker_id": "recruitment_v1",
  "status": "done",
  "result": {
    "overall_score": 85,
    "decision": "PASS",
    "reasoning": "Exceeds all technical requirements",
    "highlights": ["7 years experience", "Bonus: Kubernetes"],
    "concerns": []
  }
}
```

---

## Example 6: Poll with Retry Loop (Bash Script)

**Automatic polling with backoff**

```bash
#!/bin/bash

WORKER_ID="recruitment_v1"
TASK_ID="550e8400-e29b-41d4-a716-446655440000"
EMAIL="ceonova@agentsmarket.com"
PASSWORD="password"

max_attempts=60
attempt=1
delay=2

while [ $attempt -le $max_attempts ]; do
    echo "[Attempt $attempt/$max_attempts] Polling task..."

    response=$(curl -s -X GET \
        "http://localhost:8000/api/v1/workers/$WORKER_ID/tasks/$TASK_ID" \
        -u "$EMAIL:$PASSWORD")

    status=$(echo "$response" | grep -o '"status":"[^"]*"' | head -1 | cut -d'"' -f4)

    echo "Status: $status"

    if [ "$status" = "done" ] || [ "$status" = "failed" ]; then
        echo "Task complete!"
        echo "$response" | jq .
        exit 0
    fi

    if [ $attempt -lt $max_attempts ]; then
        echo "Waiting ${delay}s before retry..."
        sleep $delay
        delay=$((delay * 2))
        if [ $delay -gt 30 ]; then
            delay=30
        fi
    fi

    attempt=$((attempt + 1))
done

echo "Timeout: max attempts reached"
exit 1
```

**Save as `poll_task.sh` and run:**
```bash
chmod +x poll_task.sh
./poll_task.sh
```

---

## Example 7: Verbose Debug

**See all request/response details**

```bash
curl -v -X POST http://localhost:8000/api/v1/workers/recruitment_v1/tasks \
  -u "ceonova@agentsmarket.com:password" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Debug Test",
    "objective": "Test",
    "external_user_id": "test-user"
  }'
```

**Output shows:**
```
> POST /api/v1/workers/recruitment_v1/tasks HTTP/1.1
> Host: localhost:8000
> Authorization: Basic Y2Vvbm92YUBhZ2VudHNtYXJrZXQuY29tOnBhc3N3b3Jk
> Content-Type: application/json
>
< HTTP/1.1 202 Accepted
< content-type: application/json
<
{
  "task_id": "...",
  "worker_id": "..."
}
```

---

## Example 8: Error Handling

### 401 Unauthorized (Bad Credentials)

```bash
curl -X GET http://localhost:8000/api/v1/workers/ \
  -u "wrong@email.com:wrongpassword"
```

**Output:**
```json
{
  "detail": "Invalid credentials"
}
```

**Fix:** Verify email/password spelling.

### 404 Not Found (Unknown Worker)

```bash
curl -X GET http://localhost:8000/api/v1/workers/unknown_worker \
  -u "ceonova@agentsmarket.com:password"
```

**Output:**
```json
{
  "detail": "Worker not found"
}
```

**Fix:** List workers first: `GET /api/v1/workers/`

### 429 Rate Limited

```bash
curl -X POST http://localhost:8000/api/v1/workers/recruitment_v1/tasks \
  -u "ceonova@agentsmarket.com:password" \
  -H "Content-Type: application/json" \
  -d '{...}' \
  -w "\nStatus: %{http_code}\n"
```

**If 429:**
```
Status: 429
{
  "detail": "Rate limit exceeded"
}
```

**Fix:** Wait 1 minute, then retry with exponential backoff.

---

## Example 9: Using Environment Variables

**Store credentials in env (safer than hardcoding)**

```bash
# Set environment variables
export CEONOVA_EMAIL="ceonova@agentsmarket.com"
export CEONOVA_PASSWORD="password"
export AGENTSMARKET_API="http://localhost:8000"

# Use in curl
curl -X GET $AGENTSMARKET_API/api/v1/workers/ \
  -u "$CEONOVA_EMAIL:$CEONOVA_PASSWORD"
```

**Or source from .env file:**

```bash
# Load .env
source .env

# Now use variables
curl -X POST $AGENTSMARKET_API/api/v1/workers/recruitment_v1/tasks \
  -u "$CEONOVA_EMAIL:$CEONOVA_PASSWORD" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

---

## Example 10: Bulk Task Submission (Bash)

**Submit multiple tasks in sequence**

```bash
#!/bin/bash

WORKER_ID="recruitment_v1"
EMAIL="ceonova@agentsmarket.com"
PASSWORD="password"
API="http://localhost:8000"

# List of candidates
candidates=(
    "Alice Johnson"
    "Bob Smith"
    "Carol White"
)

# Submit all
task_ids=()
for candidate in "${candidates[@]}"; do
    echo "Submitting task for $candidate..."
    
    response=$(curl -s -X POST "$API/api/v1/workers/$WORKER_ID/tasks" \
        -u "$EMAIL:$PASSWORD" \
        -H "Content-Type: application/json" \
        -d "{
            \"title\": \"Screen $candidate\",
            \"objective\": \"Evaluate candidate fit\",
            \"external_user_id\": \"ceonova-user-123\"
        }")
    
    task_id=$(echo "$response" | grep -o '"task_id":"[^"]*"' | head -1 | cut -d'"' -f4)
    task_ids+=("$task_id")
    echo "  → Task ID: $task_id"
done

echo ""
echo "All tasks submitted. Task IDs:"
for id in "${task_ids[@]}"; do
    echo "  - $id"
done
```

---

## Tips & Tricks

### Pretty-Print JSON
```bash
curl -s http://localhost:8000/api/v1/workers/ \
  -u "email:password" | jq .
```

### Extract Field from Response
```bash
curl -s http://localhost:8000/api/v1/workers/recruitment_v1 \
  -u "email:password" | jq '.name'
# Output: "Recruitment Screening"
```

### Save Response to File
```bash
curl -X GET http://localhost:8000/api/v1/workers/recruitment_v1/tasks/TASK_ID \
  -u "email:password" > task_result.json
```

### Test HTTPS (Production)
```bash
curl https://api.agentsmarket.com/api/v1/workers/ \
  -u "email:password"
```

### Ignore Self-Signed SSL (Dev Only)
```bash
curl -k https://localhost:8000/api/v1/workers/ \
  -u "email:password"
```

---

## Next Steps

→ **[Python Examples](02-python.md)** — Use requests library

→ **[JavaScript Examples](03-javascript.md)** — Use Fetch API

→ **[Workflows](../02-workflows.md)** — Real-world patterns
