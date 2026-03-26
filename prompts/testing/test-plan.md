---
title: QA Test Plan Generator
category: testing
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Generate a structured QA test plan for any feature or epic — covering functional testing, edge cases, performance, and acceptance criteria.

## When to Use
- Kicking off a new feature and need to define what "done" means before writing code
- Handing off a feature to a QA team and need to give them a structured test plan
- Writing a test plan for a stakeholder who needs to sign off on quality

## The Prompt

```
You are a senior QA engineer. Generate a comprehensive test plan for the following feature.

**Feature name:** {{FEATURE_NAME}}

**Feature description:**
{{FEATURE_DESCRIPTION}}

**User stories / acceptance criteria:**
{{USER_STORIES}}
(Paste your user stories in "As a [role], I want [goal], so that [reason]" format)

**Tech stack:** {{TECH_STACK}}

**Test environment:** {{TEST_ENV}}
(Examples: "staging environment at staging.example.com", "local with Docker Compose", "Postman + staging API")

**Out of scope:**
{{OUT_OF_SCOPE}}
(Things explicitly NOT being tested in this plan)

**Definition of Done:**
{{DEFINITION_OF_DONE}}
(Examples: "all critical and high tests pass", "zero P1 bugs, <5 P2 bugs")

Generate the following test plan sections:

1. **Test Summary** — 1 paragraph describing what's being tested and the test approach
2. **Test Scope** — What IS and IS NOT being tested
3. **Functional Test Cases** — Table with: ID, Test Case, Steps, Expected Result, Priority
4. **Negative / Edge Case Tests** — What happens when users do unexpected things
5. **API Tests** — If applicable: endpoint, method, payload, expected status code
6. **Performance Benchmarks** — If applicable: load targets and pass/fail criteria
7. **Regression Risk** — What existing functionality could this feature break?
8. **Acceptance Criteria Mapping** — Which test cases satisfy which user stories
9. **Exit Criteria** — Exact conditions required to ship this feature
```

## Example Input

| Variable | Value |
|---|---|
| `{{FEATURE_NAME}}` | `Password Reset Flow` |
| `{{FEATURE_DESCRIPTION}}` | `Users can reset their password via an email link. After clicking the link, they enter a new password. The link expires in 1 hour and can only be used once.` |
| `{{USER_STORIES}}` | `As a user who forgot my password, I want to receive a reset email, so that I can regain access to my account.` |
| `{{TECH_STACK}}` | `Node.js API, PostgreSQL, SendGrid email` |
| `{{TEST_ENV}}` | `Staging at staging.example.com` |
| `{{OUT_OF_SCOPE}}` | `Email delivery rate, social login flows` |
| `{{DEFINITION_OF_DONE}}` | `All critical tests pass, zero P1 or P2 bugs` |

## Example Output

```markdown
## Test Plan: Password Reset Flow

### Test Summary
This plan covers end-to-end testing of the password reset flow, from requesting a reset
email to successfully logging in with a new password. Testing focuses on the correctness
of the token lifecycle (generation, validation, expiry, single-use) and the security of
the flow against token enumeration and reuse attacks.

---

### Functional Test Cases

| ID | Test Case | Steps | Expected Result | Priority |
|----|-----------|-------|-----------------|----------|
| TC-01 | Happy path reset | 1. Click "Forgot password" 2. Enter valid email 3. Click link in email 4. Enter new password | Login succeeds with new password | P1 Critical |
| TC-02 | Expired token | Wait 61 minutes after requesting link, click it | Show "Link expired" error, redirect to request page | P1 Critical |
| TC-03 | Token used twice | Complete reset using link, click same link again | Show "Link already used" error | P1 Critical |
| TC-04 | Invalid email address | Enter non-existent email on request form | Show generic "If this account exists..." message (no enumeration) | P1 Critical |
| TC-05 | New password too short | Enter 5-char password | Show validation error before submission | P2 High |
| TC-06 | Simultaneous reset requests | Request reset twice in 60 seconds | Second email sent, first link invalidated | P2 High |
| TC-07 | Old password still works during window | Request reset, do NOT use it, try old password | Old password still works | P2 High |
| TC-08 | Old password after reset | Complete reset with new password, try old password | Old password rejected | P1 Critical |

---

### Negative / Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| Tampered token (change 1 char) | 400 Invalid token, no DB lookup |
| Extremely long email (1000 chars) | 422 Validation error |
| SQL injection in email field | Input sanitized, no DB error |
| Reset link opened in different browser | Should work — token is not session-bound |

---

### Security Checklist
- [ ] Reset response is identical whether email exists or not (prevents enumeration)
- [ ] Token is cryptographically random (not sequential ID)
- [ ] Token is single-use (DB flag set on first use)
- [ ] Token expiry enforced server-side (not just client)
- [ ] Password change invalidates all active sessions

---

### Exit Criteria
- All P1 Critical tests pass with zero failures
- All P2 High tests pass
- Security checklist: 5/5 items verified
- Regression: login and signup flows unaffected (smoke test passes)
```

## Tips
- Paste your actual user stories in `{{USER_STORIES}}` — the test plan's acceptance criteria mapping will directly trace back to them, which stakeholders love
- The "Security Checklist" section is auto-generated for auth-related features — if your feature touches auth, money, or PII, the AI will add relevant security tests automatically
- Use the test case IDs (TC-01, TC-02...) to reference specific tests in bug reports — it creates a shared vocabulary between dev and QA
