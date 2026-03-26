---
title: Structured Bug Debugger
category: coding
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Diagnose the root cause of any bug using structured, hypothesis-driven reasoning — not guessing.

## When to Use
- You've been staring at an error for more than 10 minutes with no clear cause
- The error message is cryptic or the stack trace is deep and confusing
- You suspect the bug lives in a section of code you didn't write

## The Prompt

```
You are a senior {{LANGUAGE}} engineer with deep expertise in debugging. I have a bug and I need a structured root-cause analysis — not a quick fix.

**Language / Framework:** {{LANGUAGE}}

**The code with the bug:**
```{{LANGUAGE}}
{{CODE}}
```

**The complete error message / stack trace:**
```
{{ERROR}}
```

**What I expected to happen:**
{{EXPECTED_BEHAVIOR}}

**What actually happened:**
{{ACTUAL_BEHAVIOR}}

**Things I've already tried:**
{{ALREADY_TRIED}}

Please give me:
1. **Root Cause Analysis** — The precise reason this is failing (not just "there's a null"). Explain the execution path.
2. **Why my attempted fixes didn't work** — If I listed any.
3. **The fix** — Minimal code change with explanation.
4. **Prevention** — How to prevent this class of bug in the future.
5. **Edge cases** — Are there related bugs I should check for right now?

Do not just give me the fix. I need to understand what happened.
```

## Example Input

| Variable | Value |
|---|---|
| `{{LANGUAGE}}` | `TypeScript` |
| `{{CODE}}` | `const user = users.find(u => u.id === id); console.log(user.name);` |
| `{{ERROR}}` | `TypeError: Cannot read properties of undefined (reading 'name')` |
| `{{EXPECTED_BEHAVIOR}}` | Print the user's name |
| `{{ACTUAL_BEHAVIOR}}` | Crashes with TypeError |
| `{{ALREADY_TRIED}}` | Added `console.log(id)` — it prints correctly |

## Example Output

```
**Root Cause Analysis**
`Array.prototype.find()` returns `undefined` when no element matches the predicate —
it does NOT throw an error. Your code assumes the user always exists, but when `id`
doesn't match any user in the array, `user` is `undefined`, and reading `.name` on
`undefined` throws `TypeError`.

The execution path: find() → no match → returns undefined → user.name → TypeError.

**Why your logging didn't reveal it**
`id` logs correctly, so the id value is valid. The real issue is that the `users`
array doesn't contain a user with that id at the time `find` runs.

**The Fix**
Use optional chaining and handle the case explicitly:
const user = users.find(u => u.id === id);
if (!user) {
  console.error(`User with id ${id} not found`);
  return; // or throw a domain error
}
console.log(user.name);

**Prevention**
Establish a rule: never call `.find()` and immediately access a property on the result
without a null check. Enable TypeScript's `strictNullChecks` — it would have caught this
at compile time since find() returns `T | undefined`.

**Edge cases to check**
- Anywhere else in the codebase you use `.find()` without null checking the result
- `.filter()[0]` has the same problem — also returns undefined if empty
```

## Tips
- Include your full stack trace, not just the last line — the upper frames reveal more
- The `{{ALREADY_TRIED}}` field is crucial: it prevents the AI from suggesting what you've already done
- If the bug is intermittent, say so — it changes the diagnosis completely (race condition, state mutation, etc.)
