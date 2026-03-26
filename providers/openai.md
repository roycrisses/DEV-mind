# OpenAI (GPT-4o / o1) — Provider Guide

## Overview

OpenAI's **GPT-4o** is the workhorse of AI-assisted development — fast, highly capable, and deeply integrated into every major IDE (GitHub Copilot, Cursor, Windsurf). **o1 / o3** models add multi-step reasoning for the most complex engineering tasks. If you're coding with AI daily, there's a high chance you're already using OpenAI infrastructure whether you know it or not.

---

## Strengths for Developers

| Strength | Details |
|---|---|
| **Ecosystem integration** | Native to GitHub Copilot, Cursor, Windsurf, VS Code, and the OpenAI Assistants API |
| **Speed** | GPT-4o is among the fastest frontier models — critical for tight dev iteration loops |
| **Tool use / function calling** | Best-in-class structured output and JSON mode — ideal for building AI pipelines |
| **Code interpreter** | In ChatGPT, can run Python in a sandbox — useful for data analysis, visualization, quick scripting |
| **Broad training** | Excellent knowledge of frameworks, libraries, and patterns across virtually every language |

---

## Quirks to Know

- **Sycophancy risk** — GPT-4 models have a stronger tendency to agree with you or say your code is fine when it isn't. Add: `"Be critical. Do not soften real problems. Point out what's wrong even if I seem confident."` to review prompts.
- **Role prompts matter a lot** — OpenAI responds strongly to `"You are a senior [X] engineer"` persona framing. Always include a clear role in the system prompt.
- **JSON mode is reliable** — If you need structured output (e.g., building a tool on top of this), GPT-4o's `response_format: { type: "json_object" }` is the most reliable across providers.
- **o1 is slower but smarter** — For complex algorithmic problems or multi-step debugging, switch from GPT-4o to o1 — it uses internal chain-of-thought and is significantly better at hard problems.
- **Context window is 128K** — Large but smaller than Claude's 200K. For very large codebases, prioritize what you paste.

---

## How to Adjust DevMind Prompts for OpenAI

### 1. Use the System prompt for persona

OpenAI's Chat Completions API separates system and user messages. Put your role and constraints in the system message:

```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "You are a senior TypeScript engineer specializing in performance optimization. Be direct, critical, and specific. Do not pad your responses."
    },
    {
      "role": "user",
      "content": "Review this code: ..."
    }
  ]
}
```

### 2. Force conciseness

GPT-4o can be verbose. Control output length explicitly:

```
Respond in under 300 words. Code only — no preamble.
```

Or use `max_tokens` in the API to hard-cap the response.

### 3. Use JSON mode for structured outputs

When using DevMind prompts to build automation (e.g., auto-generating changelogs in CI):

```json
{
  "response_format": { "type": "json_object" }
}
```

Add to your prompt: `"Return your response as a JSON object with keys: summary, changes, breaking_changes"`

### 4. Chain prompts for complex tasks

For the most complex DevMind prompts (e.g., full infrastructure design), break them into a chain:
1. First call: architecture overview
2. Second call: Terraform code (using the first response as context)
3. Third call: security review of the Terraform

GPT-4o handles this better than single mega-prompts.

---

## Provider-Specific Tips

**Tip 1 — Use o1/o3 for algorithmic hard problems**
For `optimize.md` prompts on complex algorithms, or `debug.md` for deeply nested async bugs, switch to `o1` or `o3-mini`. These models use internal reasoning and consistently outperform GPT-4o on problems that require multi-step logical deduction.

```
Model: o1-preview
// Note: o1 doesn't support system messages — merge role into user message
```

**Tip 2 — Exploit Code Interpreter in ChatGPT**
For the `optimize.md` prompt, you can have GPT-4o *actually benchmark* your code in its Python sandbox. Paste `{{CODE}}` and add:
```
Run this code in your code interpreter, benchmark it, then optimize it and benchmark again.
Show me the before and after timing.
```

**Tip 3 — Structured outputs for tool-building**
If you're automating DevMind prompts (e.g., pipelines that auto-generate changelogs), use the Structured Outputs feature with a Zod/JSON Schema definition to get perfectly typed responses every time — no regex parsing needed.

---

## Comparison vs Other Providers

| Capability | GPT-4o | Claude 3.5 Sonnet | Gemini 1.5 Pro | Ollama (local) |
|---|---|---|---|---|
| **Speed** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ (hardware dependent) |
| **Tool / function calling** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **Ecosystem integrations** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Complex reasoning** | ⭐⭐⭐⭐ (o1: ⭐⭐⭐⭐⭐) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Context window** | 128K | 200K | 1M | 8K–128K |
| **Cost** | Medium | Medium | Low | Free |

---

## Recommended Models (as of 2025)

| Use Case | Recommended Model |
|---|---|
| Day-to-day dev tasks (fast iteration) | `gpt-4o` |
| Hard algorithmic problems, debugging | `o1` or `o3-mini` |
| High-volume / cost-sensitive tasks | `gpt-4o-mini` |
| Structured JSON output pipelines | `gpt-4o` with `response_format` |
| Autonomous agents / tool use | `gpt-4o` with function calling |

---

## Quick API Setup

```bash
npm install openai
export OPENAI_API_KEY=sk-...
```

```typescript
import OpenAI from 'openai';
const client = new OpenAI();

const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [
    { role: 'system', content: 'You are a senior software engineer.' },
    { role: 'user', content: 'YOUR_DEVMIND_PROMPT_HERE' },
  ],
});
```

> Full docs: https://platform.openai.com/docs
