---
title: Code Explainer
category: coding
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Get a clear, layered explanation of any code — from high-level purpose down to the exact mechanics of every line.

## When to Use
- Onboarding to a new codebase and need to understand a key module fast
- Reviewing a PR with code from an unfamiliar library or pattern
- Inheriting legacy code with no documentation

## The Prompt

```
You are a senior {{LANGUAGE}} engineer and expert technical writer. I need you to explain the following code to me.

**Language / Framework:** {{LANGUAGE}}
**My experience level:** {{EXPERIENCE_LEVEL}}
(Options: "junior developer", "senior developer unfamiliar with this library", "non-technical stakeholder")

**Code to explain:**
```{{LANGUAGE}}
{{CODE}}
```

**Specific questions I have (if any):**
{{SPECIFIC_QUESTIONS}}

Please structure your explanation as follows:

### 1. The Big Picture (2-3 sentences)
What does this code do and why does it exist?

### 2. Key Concepts
List any libraries, patterns, or language features I need to understand before reading the line-by-line breakdown.

### 3. Line-by-Line Walkthrough
Go through each significant block and explain:
- What it does
- Why it was written this way (not just what — the why matters)
- Any non-obvious gotchas or assumptions baked in

### 4. Data Flow
If data is transformed, describe the input → transformation → output.

### 5. What Could Go Wrong
Highlight any hidden assumptions, potential failure points, or edge cases the code doesn't handle.
```

## Example Input

| Variable | Value |
|---|---|
| `{{LANGUAGE}}` | `JavaScript` |
| `{{EXPERIENCE_LEVEL}}` | `senior developer unfamiliar with this library` |
| `{{CODE}}` | A RxJS observable chain with `switchMap`, `catchError`, and `shareReplay` |
| `{{SPECIFIC_QUESTIONS}}` | `Why is shareReplay used here? What happens if the HTTP request fails?` |

## Example Output

```
### 1. The Big Picture
This code creates a cached, auto-refreshing stream of user data from an API.
When any component subscribes, it triggers the HTTP request; if another component
subscribes while the request is in flight, it reuses the same request rather than
making a duplicate call.

### 2. Key Concepts
- **Observable**: A stream of values over time (like a Promise but cancellable and multi-value)
- **switchMap**: Cancels the previous inner observable and switches to the new one — ideal for search autocomplete or user-triggered fetches
- **catchError**: Intercepts errors in the stream without terminating it
- **shareReplay(1)**: Multicasts the stream and caches the last 1 emission for late subscribers

### 3. Line-by-Line Walkthrough
`this.user$ = this.route.params.pipe(`
→ Starts a derived stream from the route params observable. Every time the URL params change, this pipeline re-runs.

`switchMap(params => this.userService.getUser(params.id)),`
→ Calls the API with the new id. If the user navigates to a different user before the first response arrives, switchMap CANCELS the pending HTTP request and starts a new one. This prevents race conditions.

`catchError(err => { this.error = err.message; return EMPTY; })`
→ If the API call fails, the error is stored locally and EMPTY is returned — an observable that completes immediately without emitting. This keeps the stream alive for future emissions.

`shareReplay(1)`
→ The critical caching layer. Without this, every `async` pipe in the template would create its own HTTP request. shareReplay(1) makes all subscribers share one request and gives late subscribers the cached value instantly.

### 4. Data Flow
Route params → extract id → HTTP GET /users/:id → User object → shared/cached → template

### 5. What Could Go Wrong
- **Memory leak**: shareReplay(1) without `{refCount: true}` keeps the subscription alive even when all subscribers unsubscribe. Add `shareReplay({bufferSize: 1, refCount: true})` for auto-cleanup.
- **Stale cache**: The cache never expires — if the user data changes server-side, subscribers won't see it until the route param changes.
- **Silent failures**: catchError returns EMPTY, so the template shows nothing when there's an error. The `this.error` assignment requires you to manually display the error in your template.
```

## Tips
- Always set `{{EXPERIENCE_LEVEL}}` accurately — the explanation depth changes dramatically
- Use `{{SPECIFIC_QUESTIONS}}` to focus on the parts that actually confused you; the AI will prioritize those
- For long files, paste only the function or class you care about — stay within the context window and get a deeper answer
