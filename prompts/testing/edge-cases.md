---
title: Edge Case Explorer
category: testing
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Systematically surface the edge cases, boundary conditions, and failure scenarios that developers miss — before your users find them in production.

## When to Use
- After writing a first-pass test suite and want to harden it
- When reviewing code for a feature that handles critical user input or money
- Before releasing anything that involves state changes (DB writes, payments, auth)

## The Prompt

```
You are a QA engineer and chaos engineering expert. Your job is to think like a user who wants to break things, an attacker, and a systems engineer dealing with infrastructure failures.

**Language / Framework:** {{LANGUAGE}}

**The function / feature to analyze:**
```{{LANGUAGE}}
{{CODE}}
```

**Business context:**
{{BUSINESS_CONTEXT}}
(Examples: "this processes subscription payments", "this handles user file uploads", "this is the authentication middleware")

**What I consider the happy path:**
{{HAPPY_PATH}}

Systematically identify edge cases across these dimensions:

### 1. Input Boundary Conditions
- Empty inputs, null/undefined, zero, negative numbers
- Maximum length strings, very large numbers, integer overflow
- Unicode, emoji, RTL text, SQL injection strings, script tags

### 2. State & Concurrency
- What happens if called twice simultaneously with the same input?
- What if the DB write succeeds but the subsequent action fails?
- What if state changes between validation and execution?

### 3. External Dependency Failures
- What if the database is down / returns unexpected data?
- What if a third-party API times out or returns 500?
- What if a network call succeeds but returns malformed JSON?

### 4. Resource Limits
- What happens with very large payloads?
- What if disk/memory is exhausted mid-operation?
- What if the operation exceeds a time limit?

### 5. Security
- Can a user access another user's data by manipulating an ID?
- Can rate limiting be bypassed?
- Are there information leaks in error messages?

For each edge case found:
- **Name it** — Short label
- **Describe it** — What exact condition triggers it
- **Current behavior** — What happens RIGHT NOW (if you can tell from the code)
- **Expected behavior** — What SHOULD happen
- **Severity** — Critical / High / Medium / Low
- **Test case** — The exact test code or test description to add
```

## Example Input

| Variable | Value |
|---|---|
| `{{LANGUAGE}}` | `Python` |
| `{{CODE}}` | A `transfer_funds(from_account_id, to_account_id, amount)` function |
| `{{BUSINESS_CONTEXT}}` | `This moves money between user accounts — incorrect results have direct financial impact` |
| `{{HAPPY_PATH}}` | `Both accounts exist, from-account has sufficient balance, amount is positive` |

## Example Output

```
### 🔴 Critical

**Negative Amount Transfer**
- Condition: `amount = -100`
- Current: Deducts from destination, adds to source (reversal!)
- Expected: Raise `InvalidAmountError`
- Test: `transfer_funds(a1, a2, -100)` → expect ValueError

**Race Condition on Balance Check**
- Condition: Two concurrent transfers both read balance as sufficient, both execute
- Current: Balance goes negative (double spend)
- Expected: Use DB transaction with SELECT FOR UPDATE to lock the row
- Severity: Critical
- Test: Spawn 10 threads, each calling transfer with the full balance simultaneously

**Integer Overflow**
- Condition: `amount = 9_999_999_999_999_999` on a system using float
- Current: Floating-point rounding errors corrupt the transfer amount
- Expected: Use Decimal type for all monetary values
- Test: `transfer_funds(a1, a2, Decimal('9999999.99'))` → assert exact balance

### 🟡 High

**Same Account Transfer**
- Condition: `from_account_id == to_account_id`
- Current: Balance unchanged but operation "succeeds" — confusing audit trail
- Expected: Raise `SameAccountError` before any DB operations
- Test: `transfer_funds(same_id, same_id, 100)` → expect SameAccountError

**Non-existent Recipient**
- Condition: `to_account_id = 99999` doesn't exist in DB
- Current: DB foreign key error bubbles up as a 500 — leaks DB schema in message
- Expected: Catch and return `AccountNotFoundError` with safe message

### 🟢 Medium

**Zero Amount**
- Condition: `amount = 0`
- Current: Creates a zero-amount transaction record — pollutes history
- Expected: Reject with `InvalidAmountError("Amount must be positive")`

**Decimal Precision**
- Condition: `amount = 10.001` (3 decimal places for a currency with 2)
- Current: Silently rounds, no indication to caller
- Expected: Raise error or round with explicit caller acknowledgment
```

## Tips
- This prompt is most valuable for code that handles **money, auth, or irreversible actions** — run it before every release on those code paths
- Use `{{BUSINESS_CONTEXT}}` to frame the severity — "this lists blog posts" vs "this processes payments" triggers very different levels of scrutiny
- After the AI identifies edge cases, paste them back into the unit-tests prompt to generate the actual test code for each case
