---
title: Safe Code Refactor
category: coding
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Refactor messy, brittle, or overly complex code while guaranteeing identical external behavior.

## When to Use
- Code is working but unreadable — you dread touching it
- A function has grown beyond 50 lines and does too many things
- You're about to add a feature but the existing code makes it hard

## The Prompt

```
You are a senior {{LANGUAGE}} engineer specializing in clean code and refactoring. I need you to refactor the following code.

**Language / Framework:** {{LANGUAGE}}

**Refactoring Goal:** {{GOAL}}
(Examples: "improve readability", "reduce duplication", "apply single responsibility principle", "make it testable")

**Code to refactor:**
```{{LANGUAGE}}
{{CODE}}
```

**Constraints:**
- The external behavior must be IDENTICAL — same inputs must produce same outputs
- Do NOT add new features
- Keep the same {{LANGUAGE}} version — no new dependencies unless I approve
- {{ADDITIONAL_CONSTRAINTS}}

Please provide:
1. **What's wrong** — A brief diagnosis of why this code needs refactoring
2. **Refactoring strategy** — The approach you'll take (e.g., extract function, decompose conditional, introduce parameter object)
3. **Refactored code** — The complete refactored version with comments where necessary
4. **What changed** — A concise before/after summary of every change made
5. **Test recommendations** — How I can verify behavior hasn't changed
```

## Example Input

| Variable | Value |
|---|---|
| `{{LANGUAGE}}` | `Python` |
| `{{GOAL}}` | `Reduce nesting and improve readability` |
| `{{CODE}}` | A 40-line function with 4 levels of nested if-statements |
| `{{ADDITIONAL_CONSTRAINTS}}` | `Must stay compatible with Python 3.9` |

## Example Output

```
**What's Wrong**
This function violates the single-responsibility principle and has 4 levels of nesting
("arrow code" pattern). It's doing validation, transformation, and I/O all in one body,
making it impossible to test any part in isolation.

**Refactoring Strategy**
1. Early-return guard clauses to eliminate nesting (replace nested ifs with flat structure)
2. Extract validation into `_validate_input()`
3. Extract transformation into `_transform_record()`
4. The main function becomes an orchestrator — 8 lines

**Refactored Code**
def process_record(record: dict) -> dict | None:
    if not _validate_input(record):
        return None
    return _transform_record(record)

def _validate_input(record: dict) -> bool:
    return bool(record.get("id") and record.get("name") and record.get("email"))

def _transform_record(record: dict) -> dict:
    return {
        "id": record["id"],
        "display_name": record["name"].strip().title(),
        "contact": record["email"].lower(),
    }

**What Changed**
- 3 nested if blocks → 2 flat helper functions
- 40 lines → 15 lines across 3 functions
- Magic string access → validated access via .get()

**Test Recommendations**
Run your existing test suite. If you have none, add:
- process_record({}) → None
- process_record({"id": 1, "name": "  john  ", "email": "JOHN@X.COM"}) → {"id": 1, "display_name": "John", "contact": "john@x.com"}
```

## Tips
- Be specific in `{{GOAL}}` — "make it better" gives vague results; "apply guard clauses to reduce nesting" gives surgical results
- Paste the whole function, not just the problematic lines — the AI needs context
- If you have existing tests, mention it — the AI will tell you which ones should still pass unchanged
