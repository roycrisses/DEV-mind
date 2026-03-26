---
title: Senior Engineer Code Review
category: coding
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Get a thorough, opinionated code review from the perspective of a senior engineer — covering correctness, security, performance, and maintainability.

## When to Use
- Before opening a PR and you want to self-review first
- When you're the only developer and have no one to review your code
- When a PR comes in from a junior dev and you want a structured review checklist

## The Prompt

```
You are a senior {{LANGUAGE}} engineer with {{REVIEW_FOCUS}} expertise, doing a thorough code review. Be direct, specific, and opinionated — don't soften real problems.

**Language / Framework:** {{LANGUAGE}}
**Review Focus:** {{REVIEW_FOCUS}}
(Examples: "security and correctness", "performance and scalability", "architecture and maintainability", "all of the above")

**Context (what this code does):**
{{CONTEXT}}

**Code to review:**
```{{LANGUAGE}}
{{CODE}}
```

**Review this code across the following dimensions:**

### 🔴 Critical Issues (must fix before merge)
Bugs, security vulnerabilities, data loss risks, or correctness problems. Show the exact line and the fix.

### 🟡 Major Issues (should fix)
Performance problems, architectural mistakes, poor error handling, missing edge case coverage.

### 🟢 Minor Issues (nice to fix)
Naming, style, redundancy, minor readability concerns.

### 💡 Suggestions (optional improvements)
Things that would make this code meaningfully better but aren't required.

### ✅ What's Done Well
At least 2 things done correctly — good reviews acknowledge what works.

**Summary**
A one-paragraph verdict: is this mergeable as-is? What's the top priority to fix?
```

## Example Input

| Variable | Value |
|---|---|
| `{{LANGUAGE}}` | `Node.js / Express` |
| `{{REVIEW_FOCUS}}` | `security and correctness` |
| `{{CONTEXT}}` | `Login endpoint that authenticates users and returns a JWT` |
| `{{CODE}}` | An Express route that queries the DB with string interpolation and returns the raw password hash in the response |

## Example Output

```
### 🔴 Critical Issues

**Line 8 — SQL Injection vulnerability**
`db.query("SELECT * FROM users WHERE email = '" + req.body.email + "'")`
This is a textbook SQL injection. An attacker inputs `' OR '1'='1` and bypasses auth.
Fix: Use parameterized queries:
`db.query("SELECT * FROM users WHERE email = $1", [req.body.email])`

**Line 14 — Password hash exposed in response**
`res.json(user)` sends the entire user object including `password_hash`.
Fix: Destructure and exclude sensitive fields:
`const { password_hash, ...safeUser } = user; res.json(safeUser);`

### 🟡 Major Issues

**No rate limiting** — This endpoint has no rate limiting. An attacker can brute-force
passwords indefinitely. Add `express-rate-limit` at a minimum: 5 attempts per 15min per IP.

**JWT secret hardcoded** — `jwt.sign(payload, "mysecret123")` — this secret is in your
source code. Use `process.env.JWT_SECRET` and validate it's set at startup.

**Missing error handling** — If `db.query()` throws, the unhandled promise rejection
will crash the server in Node versions <15. Wrap in try/catch.

### 🟢 Minor Issues
- `req.body.email` is used but never validated — add email format validation
- `console.log(user)` on line 11 will log user data to production logs — remove it

### 💡 Suggestions
- Consider returning a refresh token alongside the access token for better UX
- Add request logging with a correlation ID for debugging auth failures in production

### ✅ What's Done Well
- Correct use of `bcrypt.compare()` for password verification — not rolling your own comparison
- JWT expiry is set (`expiresIn: '1h'`) — good practice

**Summary**
Do NOT merge this as-is. There are two critical security vulnerabilities: SQL injection
and password hash exposure in the response. Fix those two before anything else.
The rate limiting and JWT secret issues are important but secondary.
```

## Tips
- The `{{CONTEXT}}` field is crucial — a login endpoint has very different review criteria than a data processing job
- Use `"security and correctness"` for any code that handles auth, payments, or user data — it will be much more thorough
- If you want the AI to focus on a specific concern you already have, add it as a note after `{{CODE}}`: "I'm specifically worried about thread safety here"
