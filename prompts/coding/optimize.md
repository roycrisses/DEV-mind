---
title: Performance Optimizer
category: coding
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Identify and eliminate performance bottlenecks in any function — with measured trade-off analysis, not just micro-optimizations.

## When to Use
- A function is measurably slow under real load (profiler data preferred)
- You're hitting scalability limits and the function is the hot path
- Code review flagged O(n²) complexity or excessive memory usage

## The Prompt

```
You are a performance engineering expert in {{LANGUAGE}}. I need you to optimize the following code for {{OPTIMIZATION_TARGET}}.

**Language / Framework:** {{LANGUAGE}}
**Optimization Target:** {{OPTIMIZATION_TARGET}}
(Examples: "CPU time", "memory usage", "I/O operations", "database query count", "cold start latency")

**Current performance (if measured):**
{{CURRENT_METRICS}}
(Examples: "takes 4.2s for 10k records", "uses 800MB RAM at peak", "makes 120 DB queries per request")

**Input scale / context:**
{{SCALE_CONTEXT}}
(Examples: "processes up to 1M records", "called 500 times per second", "runs in a 128MB Lambda function")

**Code to optimize:**
```{{LANGUAGE}}
{{CODE}}
```

**Constraints:**
- {{CONSTRAINTS}}
(Examples: "cannot change the DB schema", "must remain readable for junior devs", "no new dependencies")

Please provide:
1. **Performance diagnosis** — Where is time/memory being wasted? Identify exact bottlenecks with Big-O analysis.
2. **Optimization options** — List 2-3 approaches, ranked by impact, each with trade-offs (readability, complexity, memory vs speed).
3. **Optimized code** — The best option implemented, with inline comments explaining each optimization decision.
4. **Expected gains** — Estimated improvement with reasoning.
5. **How to measure** — The exact benchmark or profiling code I should run to confirm the improvement.
```

## Example Input

| Variable | Value |
|---|---|
| `{{LANGUAGE}}` | `Python` |
| `{{OPTIMIZATION_TARGET}}` | `CPU time` |
| `{{CURRENT_METRICS}}` | `18.3 seconds for 50,000 records` |
| `{{SCALE_CONTEXT}}` | `Runs nightly, input grows 20% per month` |
| `{{CODE}}` | A loop that calls a database inside another loop (N+1 problem) |
| `{{CONSTRAINTS}}` | `Cannot change the DB schema` |

## Example Output

```
**Performance Diagnosis**
This is a classic N+1 query problem. The outer loop iterates over `orders` (N items),
and for each order, the inner call fetches the customer record — 1 query per order.
For 50,000 orders, that's 50,001 database round-trips.

Complexity: O(N) iterations × O(1) per DB call = O(N) DB queries → the bottleneck is
network/DB latency, not CPU. At ~0.3ms per query, 50,000 queries ≈ 15s. The math matches.

**Optimization Options**

Option A — Batch fetch (HIGH impact, easy): Collect all customer IDs first, fetch in
one query with WHERE id IN (...), build a lookup dict. Reduces 50,001 queries → 2.
Trade-off: loads all customers into memory (~50k objects).

Option B — JOIN in SQL (HIGH impact, cleanest): Rewrite as a single JOIN query that
fetches orders + customer data together. Zero in-memory join logic.
Trade-off: more complex SQL, but best performance at scale.

Option C — Caching (MEDIUM impact): Cache customer lookups with an LRU cache.
Trade-off: stale data risk; gains depend on customer ID reuse rate.

**Optimized Code (Option A — most practical given your constraints)**
# Fetch all customer IDs in one shot
customer_ids = [order.customer_id for order in orders]
customers = {c.id: c for c in db.query(Customer).filter(Customer.id.in_(customer_ids)).all()}
# Now iterate with zero DB calls
for order in orders:
    customer = customers[order.customer_id]  # O(1) dict lookup
    process(order, customer)

**Expected Gains**
50,001 DB queries → 2 DB queries. At 0.3ms/query: 15s → ~5ms of DB time.
Total runtime: 18.3s → ~3-4s (remaining time is Python iteration).

**How to Measure**
import time
start = time.perf_counter()
run_your_function()
print(f"Elapsed: {time.perf_counter() - start:.3f}s")
# Or with SQLAlchemy query counting: set echo=True in engine config
```

## Tips
- Always provide `{{CURRENT_METRICS}}` — without a baseline, "optimize" is meaningless and the AI will make assumptions
- Specify `{{SCALE_CONTEXT}}` — an optimization for 100 records may be wrong for 10 million
- If you have profiler output (e.g., `cProfile`, `py-spy`, Chrome DevTools flame graph), paste it — it instantly pinpoints the real bottleneck
