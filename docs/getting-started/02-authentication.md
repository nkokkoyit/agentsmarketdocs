# Authentication

**Two methods to authenticate with AgentsMarket API**

---

## Overview

AgentsMarket `/api/v1/workers/` endpoints support **two authentication methods**:

1. **JWT Bearer Token** — Time-limited, OAuth2-style (good for humans & browsers)
2. **HTTP Basic Auth** — Simple, credentials-based (good for service-to-service)

Both grant the same access. Choose based on your use case.

---

## Method 1: HTTP Basic Auth (Recommended for CEONOVA)

### How It Works

1. Your email & password are encoded in Base64
2. Sent in `Authorization: Basic` header with every request
3. No token expiry — credentials work indefinitely (until password rotated)

### Obtaining Credentials

Contact your AgentsMarket administrator to:
1. Create an account with your email
2. Set a secure password
3. Provide you both values

**Example:**
- Email: `ceonova@agentsmarket.com`
- Password: `your-secure-password`

### Using Basic Auth with cURL

**Automatic (easier):**
```bash
curl -X GET http://localhost:8000/api/v1/workers/ \
  -u "ceonova@agentsmarket.com:your-secure-password"
```

**Manual (if needed):**
```bash
# 1. Encode credentials in base64
echo -n "ceonova@agentsmarket.com:your-secure-password" | base64
# Output: Y2Vvbm92YUBhZ2VudHNtYXJrZXQuY29tOnlvdXItc2VjdXJlLXBhc3N3b3Jk

# 2. Use in header
curl -X GET http://localhost:8000/api/v1/workers/ \
  -H "Authorization: Basic Y2Vvbm92YUBhZ2VudHNtYXJrZXQuY29tOnlvdXItc2VjdXJlLXBhc3N3b3Jk"
```

### Using Basic Auth in Code

**Python (requests library):**
```python
import requests

AUTH = ("ceonova@agentsmarket.com", "your-secure-password")

response = requests.get(
    "http://localhost:8000/api/v1/workers/",
    auth=AUTH
)

print(response.json())
```

**Python (manual header):**
```python
import base64
import requests

email = "ceonova@agentsmarket.com"
password = "your-secure-password"
credentials = base64.b64encode(f"{email}:{password}".encode()).decode()

headers = {
    "Authorization": f"Basic {credentials}"
}

response = requests.get(
    "http://localhost:8000/api/v1/workers/",
    headers=headers
)

print(response.json())
```

**JavaScript (Fetch API):**
```javascript
const email = "ceonova@agentsmarket.com";
const password = "your-secure-password";
const credentials = btoa(`${email}:${password}`);

const response = await fetch("http://localhost:8000/api/v1/workers/", {
  headers: {
    "Authorization": `Basic ${credentials}`
  }
});

const data = await response.json();
console.log(data);
```

---

## Method 2: JWT Bearer Token

### How It Works

1. Exchange email + password for a JWT token
2. Use token in `Authorization: Bearer` header
3. Token expires after 30 minutes (need to refresh/re-login)

### Obtaining a Bearer Token

```bash
curl -X POST http://localhost:8000/api/v1/auth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=ceonova@agentsmarket.com&password=your-secure-password"
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJjZW9ub3ZhQGFnZW50c21hcmtldC5jb20iLCJleHAiOjE3NDg1MzIwMDB9.abc123xyz",
  "token_type": "bearer"
}
```

### Using Bearer Token

```bash
curl -X GET http://localhost:8000/api/v1/workers/ \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJjZW9ub3ZhQGFnZW50c21hcmtldC5jb20iLCJleHAiOjE3NDg1MzIwMDB9.abc123xyz"
```

### Token Expiry & Refresh

Tokens expire after **30 minutes**. When you get a 401:

```bash
# Token expired, obtain a new one
curl -X POST http://localhost:8000/api/v1/auth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=ceonova@agentsmarket.com&password=your-secure-password"

# Use new token in next request
```

### Python Example with Token Refresh

```python
import requests
import json
from datetime import datetime, timedelta

class AgentsMarketClient:
    def __init__(self, email, password, base_url="http://localhost:8000"):
        self.email = email
        self.password = password
        self.base_url = base_url
        self.token = None
        self.token_expiry = None

    def get_token(self):
        """Obtain or refresh JWT token"""
        response = requests.post(
            f"{self.base_url}/api/v1/auth/token",
            data={
                "username": self.email,
                "password": self.password
            }
        )
        data = response.json()
        self.token = data["access_token"]
        # Assume 30-minute expiry
        self.token_expiry = datetime.now() + timedelta(minutes=30)
        return self.token

    def ensure_token(self):
        """Refresh token if needed"""
        if not self.token or datetime.now() >= self.token_expiry:
            self.get_token()

    def get_workers(self):
        """List workers (with auto token refresh)"""
        self.ensure_token()

        response = requests.get(
            f"{self.base_url}/api/v1/workers/",
            headers={"Authorization": f"Bearer {self.token}"}
        )
        return response.json()

# Usage
client = AgentsMarketClient(
    email="ceonova@agentsmarket.com",
    password="your-secure-password"
)

workers = client.get_workers()
print(workers)
```

---

## Comparison: Basic Auth vs Bearer Token

| Feature | Basic Auth | Bearer Token |
|---------|-----------|-------------|
| **Setup** | Just email + password | Email + password, exchange for token |
| **Expiry** | No (until password rotated) | 30 minutes |
| **Refresh** | None | Obtain new token |
| **Use Case** | Service-to-service | Humans, browsers, apps |
| **Complexity** | Simple | Moderate |
| **Security** | Good (use HTTPS) | Better (time-limited) |

---

## Security Best Practices

### 1. Never Hardcode Credentials

**❌ Bad:**
```python
import requests
response = requests.get(
    "http://localhost:8000/api/v1/workers/",
    auth=("ceonova@agentsmarket.com", "password123")  # DON'T DO THIS
)
```

**✅ Good:**
```python
import os
import requests

email = os.getenv("CEONOVA_EMAIL")
password = os.getenv("CEONOVA_PASSWORD")

response = requests.get(
    "http://localhost:8000/api/v1/workers/",
    auth=(email, password)
)
```

### 2. Use HTTPS in Production

Basic Auth sends credentials in the `Authorization` header. Always use HTTPS to encrypt the header.

```bash
# ❌ Insecure (HTTP)
curl -u "user@example.com:password" http://api.example.com/workers/

# ✅ Secure (HTTPS)
curl -u "user@example.com:password" https://api.example.com/workers/
```

### 3. Rotate Credentials Periodically

Change your password every **3-6 months** to reduce risk of compromise.

```bash
# Contact AgentsMarket admin to change password
# (API endpoint for password change TBD)
```

### 4. Mask Credentials in Logs

When logging requests, don't include auth headers:

**❌ Bad:**
```python
print(f"Request: {response.request.headers}")  # Shows auth header!
```

**✅ Good:**
```python
import copy
headers = copy.copy(response.request.headers)
headers.pop("Authorization", None)  # Remove auth header before logging
print(f"Request headers: {headers}")
```

### 5. Don't Expose Credentials in Frontend

If building a web UI, never send credentials from browser to AgentsMarket. Instead:

1. Browser sends credentials to your CEONOVA backend
2. Backend authenticates & verifies
3. Backend calls AgentsMarket API on behalf of user
4. Backend returns results to browser

```
Browser → CEONOVA Backend (credentials) → AgentsMarket API
```

---

## Environment Variable Setup

### Using .env File (Development)

Create `.env` at project root:

```env
CEONOVA_EMAIL=ceonova@agentsmarket.com
CEONOVA_PASSWORD=your-secure-password
AGENTSMARKET_API_BASE=http://localhost:8000
```

Load in code:

```python
import os
from dotenv import load_dotenv

load_dotenv()

email = os.getenv("CEONOVA_EMAIL")
password = os.getenv("CEONOVA_PASSWORD")
api_base = os.getenv("AGENTSMARKET_API_BASE")
```

### Using Shell Environment (Production)

```bash
export CEONOVA_EMAIL=ceonova@agentsmarket.com
export CEONOVA_PASSWORD=your-secure-password
export AGENTSMARKET_API_BASE=https://api.agentsmarket.com

python your_script.py
```

### Using Docker

In `docker-compose.yml`:

```yaml
services:
  ceonova-integration:
    image: ceonova-app:latest
    environment:
      CEONOVA_EMAIL: ${CEONOVA_EMAIL}
      CEONOVA_PASSWORD: ${CEONOVA_PASSWORD}
      AGENTSMARKET_API_BASE: ${AGENTSMARKET_API_BASE}
```

Run:

```bash
CEONOVA_EMAIL=... CEONOVA_PASSWORD=... docker-compose up
```

---

## Testing Authentication

### Verify Credentials Work

```bash
# Test HTTP Basic Auth
curl -v -u "ceonova@agentsmarket.com:your-secure-password" \
  http://localhost:8000/api/v1/workers/

# Look for:
# > Authorization: Basic ...
# < 200 OK (success)
# < 401 Unauthorized (bad credentials)
```

### Debug Failed Authentication

```bash
# Verbose output shows headers
curl -v -u "ceonova@agentsmarket.com:password" \
  http://localhost:8000/api/v1/workers/

# Check:
# 1. Email spelling (case-sensitive)
# 2. Password correctness
# 3. HTTPS in production (not HTTP)
# 4. Authorization header format

# If 401 Unauthorized:
# - Try bearer token approach
# - Contact AgentsMarket support to verify account exists
```

---

## What's Next?

→ **[API Reference](../api-guide/01-api-overview.md)** — Learn all available endpoints

→ **[Code Examples](../integration-patterns/code-examples/01-curl.md)** — Ready-to-use samples

→ **[Troubleshooting](../operations/01-troubleshooting.md)** — Solve common issues
