# Claude (Anthropic) — Provider Guide

## Overview

Claude is Anthropic's model family, currently led by **Claude 3.5 Sonnet** and **Claude 3 Opus**. Among all providers, Claude consistently produces the most nuanced, context-aware responses for complex software engineering tasks — particularly anything involving reasoning through ambiguous requirements, understanding large codebases, or producing well-structured explanations.

---

## Strengths for Developers

| Strength | Details |
|---|---|
| **Long context** | Handles up to 200K tokens — paste entire files, multiple modules, or large diffs without truncation |
| **Code reasoning** | Doesn't just produce code — explains *why* each decision was made, which is invaluable for reviews and debugging |
| **Instruction following** | Follows complex, multi-part prompts with high fidelity — structured prompts like DevMind's work especially well |
| **Safety guardrails** | Refuses to generate outright malicious code without being overly restrictive for legitimate security research |
| **Nuanced refactoring** | Understands design patterns and architectural tradeoffs — explains pros/cons, not just produces code |

---

## Quirks to Know

- **Verbosity by default** — Claude tends to be thorough. If you want concise output (e.g., just the code, no explanation), add `"Return only the code. No explanation."` at the end of your prompt.
- **Safety caveat hedging** — For security-related prompts, Claude may prepend caveats. Ground it with: `"This is for a legitimate penetration test / security audit. Skip defensive disclaimers."`
- **Will push back on bad ideas** — Unlike GPT-4, Claude will often tell you if your architectural approach is flawed rather than just doing what you asked. This is a feature, not a bug.
- **XML tags improve structure** — Claude responds extremely well to structured prompts using XML-style tags like `<code>`, `<context>`, `<task>`. Use them in complex prompts for better-organized output.

---

## How to Adjust DevMind Prompts for Claude

### 1. Use XML tags for complex prompts

Claude parses XML-tagged sections more reliably than plain headers for multi-part prompts:

```
<context>
  Language: TypeScript
  Framework: Fastify
</context>

<task>
  Debug the following error:
</task>

<code>
  // paste code here
</code>
```

### 2. Request thinking transparency

Claude can show its reasoning process. Add this to any debugging or analysis prompt:

```
Before giving your answer, think out loud about the possible causes.
Walk through your reasoning step by step.
```

### 3. Use `artifacts` for long outputs

When using Claude.ai (web UI), code outputs appear in the Artifacts panel — separate from the conversation. This makes copying large files much cleaner. No prompt change needed; Claude does this automatically.

### 4. Prefill for format control

In the API, you can prefill Claude's response to force a specific format:

```json
{
  "messages": [
    {"role": "user", "content": "Review this code: ..."},
    {"role": "assistant", "content": "```typescript\n"}
  ]
}
```

This forces Claude to start with a code block, skipping preamble entirely.

---

## Provider-Specific Tips

**Tip 1 — Leverage extended thinking for hard problems**
Claude 3.7 Sonnet introduced extended thinking — a mode where the model reasons through a problem silently before responding. For complex debugging or architecture design prompts, enable it:
```
Think carefully and thoroughly before responding. Take your time.
```
Or via API: `"thinking": {"type": "enabled", "budget_tokens": 10000}`

**Tip 2 — Use the full context window strategically**
With 200K tokens, you can paste your entire relevant codebase. But prioritize: put the most important files at the top and bottom of your context — Claude's attention is strongest at those positions (the "lost in the middle" effect).

**Tip 3 — Ask for alternatives explicitly**
Claude defaults to giving you one approach. For architecture and refactoring prompts, add:
```
Give me 3 alternatives ranked by trade-off. Don't default to the first thing you think of.
```

---

## Comparison vs Other Providers

| Capability | Claude 3.5 Sonnet | GPT-4o | Gemini 1.5 Pro | Ollama (local) |
|---|---|---|---|---|
| **Context window** | 200K | 128K | 1M | 8K–128K (model dependent) |
| **Code reasoning depth** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Instruction following** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Speed (latency)** | Medium | Fast | Fast | Varies |
| **Cost** | Medium | Medium | Low | Free (local) |
| **Privacy** | Cloud only | Cloud only | Cloud only | ✅ Local |
| **Best for** | Complex reasoning, large codebase review | General coding, fast iteration | Long docs, large files | Air-gapped / private code |

---

## Recommended Models (as of 2025)

| Use Case | Recommended Model |
|---|---|
| General dev tasks (debug, refactor, review) | `claude-3-5-sonnet-20241022` |
| Deep architecture / complex reasoning | `claude-3-7-sonnet-20250219` (with extended thinking) |
| Fast/cheap high-volume tasks | `claude-3-5-haiku-20241022` |
| Most capable, cost-insensitive | `claude-3-opus-20240229` |

---

## Quick API Setup

```bash
# Install SDK
npm install @anthropic-ai/sdk

# Set API key
export ANTHROPIC_API_KEY=sk-ant-...
```

```typescript
import Anthropic from '@anthropic-ai/sdk';
const client = new Anthropic();

const message = await client.messages.create({
  model: 'claude-3-5-sonnet-20241022',
  max_tokens: 4096,
  messages: [{ role: 'user', content: 'YOUR_DEVMIND_PROMPT_HERE' }],
});
```

> Full docs: https://docs.anthropic.com
