---
title: API Endpoint Documenter
category: docs
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Generate comprehensive API documentation for any endpoint — including request/response schemas, error codes, authentication details, and runnable curl examples.

## When to Use
- You've built endpoints with no documentation and need to catch up fast
- Onboarding a new developer who needs to understand your API immediately
- Preparing an API for external/public consumption

## The Prompt

```
You are a technical writer specializing in API documentation. Generate complete, developer-friendly documentation for the following API endpoint.

**Endpoint:** {{METHOD}} {{PATH}}
(Examples: "POST /api/v1/users", "GET /api/v2/orders/:id")

**Service name:** {{SERVICE_NAME}}

**Authentication:** {{AUTH_TYPE}}
(Examples: "Bearer JWT in Authorization header", "API Key in X-API-Key header", "none — public endpoint")

**What this endpoint does:**
{{DESCRIPTION}}

**Request body / query params:**
{{REQUEST_SCHEMA}}
(List each field with name, type, required/optional, and a description)

**Successful response:**
{{RESPONSE_SCHEMA}}
(List each field in the response object)

**Possible error cases:**
{{ERROR_CASES}}
(Examples: "404 if user not found", "422 if email already taken", "401 if token expired")

**Rate limiting (if any):**
{{RATE_LIMIT}}
(Examples: "100 requests per minute per API key", "none")

Generate:
1. **Endpoint summary card** — Method, path, description, auth requirement
2. **Request reference** — Headers, path params, query params, request body (with field table)
3. **Response reference** — Success response with field table, full JSON example
4. **Error reference** — Table of all error codes with cause and resolution
5. **Code examples** — curl, fetch (JavaScript), and requests (Python) — all runnable
6. **Notes & gotchas** — Any non-obvious behavior developers must know
```

## Example Input

| Variable | Value |
|---|---|
| `{{METHOD}}` | `POST` |
| `{{PATH}}` | `/api/v1/auth/login` |
| `{{SERVICE_NAME}}` | `Auth Service` |
| `{{AUTH_TYPE}}` | `None — this is the login endpoint` |
| `{{DESCRIPTION}}` | `Authenticates a user with email and password. Returns a JWT access token and refresh token.` |
| `{{REQUEST_SCHEMA}}` | `email (string, required), password (string, required, min 8 chars), remember_me (boolean, optional, default false)` |
| `{{RESPONSE_SCHEMA}}` | `access_token (string), refresh_token (string), expires_in (number, seconds), user.id, user.email, user.role` |
| `{{ERROR_CASES}}` | `401 invalid credentials, 422 email format invalid, 429 too many attempts, 500 internal error` |
| `{{RATE_LIMIT}}` | `5 attempts per 15 minutes per IP` |

## Example Output

```markdown
## POST /api/v1/auth/login

> Authenticate a user and receive access + refresh tokens.

**Auth required:** No  
**Rate limit:** 5 requests / 15 min per IP

---

### Request

**Headers**
| Header | Value | Required |
|--------|-------|----------|
| Content-Type | application/json | ✅ Yes |

**Body**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| email | string | ✅ Yes | User's email address |
| password | string | ✅ Yes | Minimum 8 characters |
| remember_me | boolean | ❌ No | Extends token TTL from 1h to 30d. Default: false |

---

### Response — 200 OK

\```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "expires_in": 3600,
  "user": {
    "id": "usr_01HX2KQMN8Y",
    "email": "user@example.com",
    "role": "admin"
  }
}
\```

---

### Errors

| Code | Meaning | Resolution |
|------|---------|------------|
| 401 | Invalid credentials | Verify email and password |
| 422 | Malformed email | Ensure email field is a valid format |
| 429 | Too many attempts | Wait 15 minutes or implement exponential backoff |
| 500 | Internal server error | Retry once; if persistent, contact support |

---

### Code Examples

**curl**
\```bash
curl -X POST https://api.example.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"mypassword"}'
\```

**JavaScript (fetch)**
\```js
const res = await fetch('https://api.example.com/api/v1/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email: 'user@example.com', password: 'mypassword' })
});
const { access_token } = await res.json();
\```

**Python (requests)**
\```python
import requests
r = requests.post('https://api.example.com/api/v1/auth/login',
    json={'email': 'user@example.com', 'password': 'mypassword'})
access_token = r.json()['access_token']
\```

---

### Notes & Gotchas
- The access token expires in **1 hour** by default. Set `remember_me: true` for a 30-day token.
- Use the `refresh_token` with `POST /api/v1/auth/refresh` to get a new access token without re-login.
- The `429` response includes a `Retry-After` header with the exact seconds to wait.
```

## Tips
- Include your actual field names in `{{REQUEST_SCHEMA}}` — this produces accurate field tables, not generic placeholders
- If you have multiple related endpoints, run this prompt for each one and combine them into a single API reference doc
- Specify the exact error codes your API uses in `{{ERROR_CASES}}` — don't leave it blank, or the AI will guess generic HTTP codes
