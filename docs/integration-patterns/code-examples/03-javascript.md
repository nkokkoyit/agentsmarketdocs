# JavaScript Examples

**Ready-to-use JavaScript samples using Fetch API**

---

## Setup

### Configuration

Create `agentsmarket.js`:

```javascript
const API_BASE = process.env.AGENTSMARKET_API || "http://localhost:8000";
const AUTH_EMAIL = process.env.CEONOVA_EMAIL || "ceonova@agentsmarket.com";
const AUTH_PASSWORD = process.env.CEONOVA_PASSWORD || "your-password";

// Helper: Generate Basic Auth header
function getBasicAuthHeader(email, password) {
  const credentials = `${email}:${password}`;
  const encoded = btoa(credentials);
  return {
    "Authorization": `Basic ${encoded}`,
    "Content-Type": "application/json"
  };
}

const authHeaders = getBasicAuthHeader(AUTH_EMAIL, AUTH_PASSWORD);
```

### Environment Setup (Node.js)

Create `.env`:
```env
AGENTSMARKET_API=http://localhost:8000
CEONOVA_EMAIL=ceonova@agentsmarket.com
CEONOVA_PASSWORD=your-password
```

Load with:
```bash
npm install dotenv
```

```javascript
require('dotenv').config();
const API_BASE = process.env.AGENTSMARKET_API;
```

---

## Example 1: Discover Workers

```javascript
async function discoverWorkers() {
  const response = await fetch(`${API_BASE}/api/v1/workers/`, {
    headers: authHeaders
  });

  if (!response.ok) {
    throw new Error(`Error: ${response.status} - ${response.statusText}`);
  }

  const data = await response.json();
  data.workers.forEach(worker => {
    console.log(`✓ ${worker.worker_id}: ${worker.name}`);
    console.log(`  Capabilities: ${worker.capabilities.join(", ")}`);
  });

  return data;
}

// Usage
try {
  await discoverWorkers();
} catch (error) {
  console.error("Failed to discover workers:", error);
}
```

---

## Example 2: Submit a Task

```javascript
async function submitTask(
  workerId,
  title,
  objective,
  externalUserId,
  context = null,
  expectedOutput = null
) {
  const payload = {
    title,
    objective,
    external_user_id: externalUserId
  };

  if (context) payload.context = context;
  if (expectedOutput) payload.expected_output = expectedOutput;

  const response = await fetch(
    `${API_BASE}/api/v1/workers/${workerId}/tasks`,
    {
      method: "POST",
      headers: authHeaders,
      body: JSON.stringify(payload)
    }
  );

  if (response.status !== 202) {
    const error = await response.text();
    throw new Error(`Submit failed (${response.status}): ${error}`);
  }

  return await response.json();
}

// Usage
try {
  const task = await submitTask(
    "recruitment_v1",
    "Screen Alice Johnson",
    "Evaluate candidate fit",
    "ceonova-user-456",
    "Senior Engineer role, 5+ years required"
  );

  console.log(`✓ Task submitted!`);
  console.log(`  Task ID: ${task.task_id}`);
  console.log(`  Worker: ${task.worker_id}`);
} catch (error) {
  console.error("Task submission failed:", error);
}
```

---

## Example 3: List User's Tasks

```javascript
async function listUserTasks(
  workerId,
  externalUserId,
  limit = 20,
  offset = 0
) {
  const params = new URLSearchParams({
    external_user_id: externalUserId,
    limit,
    offset
  });

  const response = await fetch(
    `${API_BASE}/api/v1/workers/${workerId}/tasks?${params}`,
    { headers: authHeaders }
  );

  if (!response.ok) {
    throw new Error(`Error: ${response.status}`);
  }

  return await response.json();
}

// Usage
const tasks = await listUserTasks("recruitment_v1", "ceonova-user-456");
console.log(`Total tasks: ${tasks.total}`);
tasks.items.forEach(task => {
  console.log(`  • ${task.task_id}: ${task.status}`);
});
```

---

## Example 4: Poll Task Status (Single Poll)

```javascript
async function pollTaskOnce(
  workerId,
  taskId,
  externalUserId = null
) {
  let url = `${API_BASE}/api/v1/workers/${workerId}/tasks/${taskId}`;
  
  if (externalUserId) {
    url += `?external_user_id=${externalUserId}`;
  }

  const response = await fetch(url, { headers: authHeaders });

  if (!response.ok) {
    throw new Error(`Error: ${response.status}`);
  }

  return await response.json();
}

// Usage
const task = await pollTaskOnce("recruitment_v1", "550e8400-e29b-41d4-a716-446655440000");
console.log(`Status: ${task.status}`);
if (task.status === 'done') {
  console.log(`Result:`, task.result);
}
```

---

## Example 5: Poll with Exponential Backoff

```javascript
async function pollTaskStatus(
  workerId,
  taskId,
  maxRetries = 30,
  initialDelay = 2000
) {
  let delay = initialDelay;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    console.log(`[Attempt ${attempt}/${maxRetries}] Polling task...`);

    try {
      const task = await pollTaskOnce(workerId, taskId);
      console.log(`  Status: ${task.status}`);

      // Task complete
      if (["done", "failed"].includes(task.status)) {
        return task;
      }

      // Still running, wait and retry
      if (attempt < maxRetries) {
        console.log(`  Waiting ${delay}ms before retry...`);
        await new Promise(resolve => setTimeout(resolve, delay));
        delay = Math.min(delay * 2, 30000); // Exponential backoff, cap at 30s
      }
    } catch (error) {
      console.error(`  Error: ${error.message}`);
      if (attempt === maxRetries) throw error;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw new Error(`Task polling timeout after ${maxRetries} attempts`);
}

// Usage
const task = await pollTaskStatus("recruitment_v1", "550e8400-e29b-41d4-a716-446655440000");
console.log(`\n✓ Task complete!`);
console.log(`  Final status: ${task.status}`);
if (task.status === 'done') {
  console.log(`  Result:`, task.result);
}
```

---

## Example 6: End-to-End Workflow

```javascript
async function evaluateCandidate(
  candidateName,
  candidateInfo,
  jobDescription,
  externalUserId
) {
  console.log(`Evaluating ${candidateName}...`);
  console.log("=".repeat(50));

  // Step 1: Submit task
  console.log(`\n1. Submitting task...`);
  const task = await submitTask(
    "recruitment_v1",
    `Screen ${candidateName}`,
    "Evaluate candidate fit",
    externalUserId,
    `Job: ${jobDescription}\n\nCandidate: ${candidateInfo}`,
    "Score (0-100) and recommendation (PASS/MAYBE/REJECT)"
  );

  console.log(`   ✓ Task ID: ${task.task_id}`);

  // Step 2: Poll for result
  console.log(`\n2. Waiting for result...`);
  const finalTask = await pollTaskStatus("recruitment_v1", task.task_id);

  // Step 3: Process result
  console.log(`\n3. Result:`);
  console.log(`   Status: ${finalTask.status}`);

  if (finalTask.status === 'done') {
    const result = finalTask.result;
    console.log(`   Score: ${result.overall_score}/100`);
    console.log(`   Decision: ${result.decision}`);
    console.log(`   Reasoning: ${result.reasoning}`);
  } else if (finalTask.status === 'failed') {
    console.log(`   Error: ${finalTask.result.error}`);
  }

  console.log("=".repeat(50));
  return finalTask;
}

// Usage
try {
  const result = await evaluateCandidate(
    "Alice Johnson",
    "7 years Python, 3 years Docker, 2 years Kubernetes",
    "Senior Backend Engineer - 5+ years required",
    "ceonova-user-456"
  );
} catch (error) {
  console.error("Evaluation failed:", error);
}
```

---

## Example 7: Batch Evaluation (Promise.all)

```javascript
async function evaluateCandidatesParallel(
  candidates,
  jobDescription,
  externalUserId
) {
  console.log(`Evaluating ${candidates.length} candidates in parallel...`);
  console.log("=".repeat(50));

  // Step 1: Submit all tasks
  console.log(`\n1. Submitting ${candidates.length} tasks...`);
  const submitPromises = candidates.map(candidate =>
    submitTask(
      "recruitment_v1",
      `Screen ${candidate.name}`,
      "Evaluate fit",
      externalUserId,
      `Job: ${jobDescription}\n\nCandidate: ${candidate.info}`
    ).then(task => ({
      candidate: candidate.name,
      taskId: task.task_id
    }))
  );

  const submitted = await Promise.all(submitPromises);
  console.log(`   ✓ All ${submitted.length} tasks submitted!`);

  // Step 2: Poll all tasks
  console.log(`\n2. Polling results...`);
  const pollPromises = submitted.map(({ candidate, taskId }) =>
    pollTaskStatus("recruitment_v1", taskId)
      .then(task => ({
        candidate,
        taskId,
        status: task.status,
        result: task.result
      }))
      .catch(error => ({
        candidate,
        taskId,
        status: "error",
        error: error.message
      }))
  );

  const results = await Promise.all(pollPromises);

  // Step 3: Display results
  console.log(`\n${"=".repeat(50)}`);
  console.log("RESULTS:");
  console.log("=".repeat(50));

  results.forEach(result => {
    if (result.status === 'done') {
      const r = result.result;
      console.log(`✓ ${result.candidate}: ${r.overall_score}/100 - ${r.decision}`);
    } else if (result.status === 'error') {
      console.log(`✗ ${result.candidate}: ERROR - ${result.error}`);
    } else {
      console.log(`? ${result.candidate}: ${result.status}`);
    }
  });

  return results;
}

// Usage
const candidates = [
  { name: "Alice Johnson", info: "7 yrs Python, 3 yrs Docker" },
  { name: "Bob Smith", info: "5 yrs Python, 1 yr Kubernetes" },
  { name: "Carol White", info: "10 yrs backend, AWS expert" }
];

try {
  const results = await evaluateCandidatesParallel(
    candidates,
    "Senior Backend Engineer - 5+ years Python required",
    "ceonova-team-1"
  );
} catch (error) {
  console.error("Batch evaluation failed:", error);
}
```

---

## Example 8: Error Handling

```javascript
async function safeSubmitTask(
  workerId,
  title,
  objective,
  externalUserId
) {
  try {
    return await submitTask(workerId, title, objective, externalUserId);
  } catch (error) {
    const message = error.message;

    if (message.includes("401")) {
      return {
        success: false,
        error: "Authentication failed",
        reason: "Check credentials in .env"
      };
    } else if (message.includes("404")) {
      return {
        success: false,
        error: "Worker not found",
        reason: "List workers and verify worker_id"
      };
    } else if (message.includes("422")) {
      return {
        success: false,
        error: "Invalid request",
        reason: "Check request body matches schema"
      };
    } else if (message.includes("429")) {
      return {
        success: false,
        error: "Rate limited",
        reason: "Wait 1 minute before retrying"
      };
    } else {
      return {
        success: false,
        error: "Unexpected error",
        reason: message
      };
    }
  }
}

// Usage
const result = await safeSubmitTask(
  "recruitment_v1",
  "Test",
  "Test",
  "test-user"
);

if (result.success) {
  console.log(`✓ Task submitted: ${result.task_id}`);
} else {
  console.error(`✗ ${result.error}: ${result.reason}`);
}
```

---

## Example 9: Async/Await with Try-Catch

```javascript
async function runEvaluation() {
  try {
    // Step 1: Discover workers
    console.log("Step 1: Discovering workers...");
    const workers = await discoverWorkers();
    const recruitmentWorker = workers.workers.find(
      w => w.worker_id === "recruitment_v1"
    );

    if (!recruitmentWorker) {
      throw new Error("recruitment_v1 worker not found");
    }

    // Step 2: Submit task
    console.log("\nStep 2: Submitting task...");
    const task = await submitTask(
      "recruitment_v1",
      "Screen candidate",
      "Evaluate fit",
      "ceonova-user-1"
    );

    // Step 3: Poll result
    console.log("\nStep 3: Waiting for result...");
    const finalTask = await pollTaskStatus("recruitment_v1", task.task_id);

    // Step 4: Display result
    console.log("\nStep 4: Result:");
    console.log(finalTask.result);

    return finalTask;
  } catch (error) {
    console.error("Pipeline failed:", error.message);
    throw error;
  }
}

// Usage
runEvaluation()
  .then(() => console.log("\n✓ Pipeline completed"))
  .catch(error => console.error("\n✗ Pipeline failed:", error));
```

---

## Example 10: Class-Based Client

```javascript
class AgentsMarketClient {
  constructor(apiBase, email, password) {
    this.apiBase = apiBase;
    this.email = email;
    this.password = password;
    this.authHeaders = this.createAuthHeaders();
  }

  createAuthHeaders() {
    const credentials = `${this.email}:${this.password}`;
    const encoded = btoa(credentials);
    return {
      "Authorization": `Basic ${encoded}`,
      "Content-Type": "application/json"
    };
  }

  async discoverWorkers() {
    const response = await fetch(`${this.apiBase}/api/v1/workers/`, {
      headers: this.authHeaders
    });
    if (!response.ok) throw new Error(`Error: ${response.status}`);
    return await response.json();
  }

  async submitTask(workerId, title, objective, externalUserId, context = null) {
    const payload = {
      title,
      objective,
      external_user_id: externalUserId
    };
    if (context) payload.context = context;

    const response = await fetch(
      `${this.apiBase}/api/v1/workers/${workerId}/tasks`,
      {
        method: "POST",
        headers: this.authHeaders,
        body: JSON.stringify(payload)
      }
    );

    if (response.status !== 202) throw new Error(`Submit failed: ${response.status}`);
    return await response.json();
  }

  async pollTask(workerId, taskId) {
    const response = await fetch(
      `${this.apiBase}/api/v1/workers/${workerId}/tasks/${taskId}`,
      { headers: this.authHeaders }
    );
    if (!response.ok) throw new Error(`Poll failed: ${response.status}`);
    return await response.json();
  }
}

// Usage
const client = new AgentsMarketClient(
  "http://localhost:8000",
  process.env.CEONOVA_EMAIL,
  process.env.CEONOVA_PASSWORD
);

const workers = await client.discoverWorkers();
console.log(`Available workers: ${workers.workers.length}`);

const task = await client.submitTask(
  "recruitment_v1",
  "Test",
  "Test",
  "user-1"
);
console.log(`Task submitted: ${task.task_id}`);
```

---

## Tips & Best Practices

### Handle Network Timeouts
```javascript
async function fetchWithTimeout(url, options = {}, timeout = 5000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal
    });
    return response;
  } finally {
    clearTimeout(id);
  }
}
```

### Retry Failed Requests
```javascript
async function retryFetch(url, options = {}, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fetch(url, options);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(r => setTimeout(r, 1000 * (i + 1)));
    }
  }
}
```

### Log Requests/Responses
```javascript
async function fetchWithLogging(url, options = {}) {
  console.log(`→ ${options.method || 'GET'} ${url}`);
  const response = await fetch(url, options);
  console.log(`← ${response.status}`);
  return response;
}
```

---

## Next Steps

→ **[cURL Examples](01-curl.md)** — Command-line testing

→ **[Workflows](../02-workflows.md)** — Real-world patterns

→ **[Troubleshooting](../../operations/01-troubleshooting.md)** — Debug errors
