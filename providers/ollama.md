# Ollama — Provider Guide

## Overview

**Ollama** lets you run open-source LLMs (Llama 3, Mistral, Qwen, CodeLlama, DeepSeek-Coder, and more) entirely on your own hardware — no API keys, no data leaving your machine, no per-token costs. For developers working with proprietary codebases, sensitive client data, or air-gapped environments, Ollama is the only viable option. Response quality is lower than frontier cloud models, but has improved dramatically with newer open-source models in 2024-2025.

---

## Strengths for Developers

| Strength | Details |
|---|---|
| **Complete privacy** | Zero data leaves your machine — ideal for client code, NDAs, and regulated industries |
| **No cost** | Run unlimited requests after the one-time hardware/model download investment |
| **Offline operation** | Works without internet — plane, submarine, air-gapped dev environments |
| **Model variety** | 100+ models available: general (Llama 3), code-specialized (DeepSeek-Coder, CodeLlama), small+fast (Phi-3) |
| **OpenAI-compatible API** | Drop-in replacement for OpenAI API — swap `baseURL` and you're done |

---

## Quirks to Know

- **Hardware is the bottleneck** — Quality and speed depend entirely on your GPU/RAM. A 70B model on a MacBook M3 Pro runs at ~15 tokens/sec; the same on a consumer GPU may be unusably slow. Match model size to hardware.
- **Smaller context windows** — Most local models cap at 8K–32K tokens (vs 128K–1M for cloud). For DevMind prompts with large code pastes, use a model with at least 8K context and trim your `{{CODE}}` if needed.
- **Instruction following degrades** — Smaller models (7B, 8B) struggle with long, complex prompts. If following breaks down, simplify the prompt or switch to a 13B+ model.
- **No safety filters** — Ollama models run without cloud-side safety guardrails. This is useful for security research prompts that cloud providers might refuse.
- **Model-specific system prompt format** — Different models (Llama vs Mistral vs Qwen) use different chat templates. Ollama handles this automatically if you use the `/api/chat` endpoint — don't use raw text completion for instruct models.

---

## How to Adjust DevMind Prompts for Ollama

### 1. Shorten prompts for smaller models

7B–13B models can't maintain coherence across very long prompts. For local models, trim DevMind prompts to the essentials:

**Original:** Full debug prompt with all 6 output sections  
**Ollama version:**
```
You are a {{LANGUAGE}} debugger.

Code: {{CODE}}
Error: {{ERROR}}

Give me:
1. Root cause (2-3 sentences)
2. The fix (code only)
```

### 2. Choose a code-specialized model

For all `prompts/coding/**` prompts, use a code-specialized model rather than a general model:

```bash
# Best options for coding tasks
ollama pull deepseek-coder-v2      # Best overall coding, 16B
ollama pull qwen2.5-coder:14b     # Strong, fast on Apple Silicon
ollama pull codellama:13b          # Good for completion-style tasks
ollama pull starcoder2:15b         # Strong on code generation
```

### 3. Use the OpenAI-compatible endpoint

Ollama exposes an OpenAI-compatible API at `http://localhost:11434/v1`. Swap it into any DevMind integration without code changes:

```typescript
import OpenAI from 'openai';

const client = new OpenAI({
  baseURL: 'http://localhost:11434/v1',
  apiKey: 'ollama', // required by the SDK but not used
});

const response = await client.chat.completions.create({
  model: 'deepseek-coder-v2',
  messages: [{ role: 'user', content: 'YOUR_DEVMIND_PROMPT_HERE' }],
});
```

### 4. Increase context window explicitly

Ollama models default to 2K–4K context. Override with `num_ctx` for longer prompts:

```bash
ollama run deepseek-coder-v2 --parameter num_ctx 16384
```

Or in the API:
```json
{
  "model": "deepseek-coder-v2",
  "options": { "num_ctx": 16384 }
}
```

---

## Provider-Specific Tips

**Tip 1 — Use a Modelfile for persistent configuration**
Instead of setting parameters every run, create a Modelfile with your preferred settings:

```dockerfile
FROM deepseek-coder-v2

PARAMETER num_ctx 16384
PARAMETER temperature 0.1
SYSTEM "You are an expert software engineer. Be concise and direct. Return code without preamble."
```

```bash
ollama create devmind-coder -f Modelfile
ollama run devmind-coder
```

**Tip 2 — Use temperature 0 for deterministic code**
For coding tasks, set `temperature: 0` (or very low, `0.1`). You want the most likely correct answer, not creative variation:
```json
{ "options": { "temperature": 0.1 } }
```

**Tip 3 — Hardware recommendations by model size**

| Model Size | Minimum RAM | Recommended GPU | Speed |
|---|---|---|---|
| 7B / 8B | 8GB RAM | Any modern GPU with 8GB VRAM | ⚡ Fast |
| 13B / 14B | 16GB RAM | RTX 3080 / M2 Pro | 🔄 Medium |
| 32B | 32GB RAM | RTX 4090 / M3 Max | 🐢 Slower |
| 70B+ | 64GB RAM | 2× RTX 4090 / Mac Studio | 🐌 Slow |

For most DevMind prompts, **13B–14B models hit the best quality/speed tradeoff**.

---

## Comparison vs Other Providers

| Capability | Ollama (local) | GPT-4o | Claude 3.5 Sonnet | Gemini 1.5 Pro |
|---|---|---|---|---|
| **Privacy** | ✅ 100% local | ❌ Cloud | ❌ Cloud | ❌ Cloud |
| **Cost** | ✅ Free | 💰 Per token | 💰 Per token | 💰 Per token |
| **Offline** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Code quality** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Context window** | ⭐⭐⭐ (8K–32K) | ⭐⭐⭐⭐ (128K) | ⭐⭐⭐⭐⭐ (200K) | ⭐⭐⭐⭐⭐ (1M) |
| **Speed** | Varies by hardware | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## Quick Setup

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh   # Linux/macOS
# Windows: download from https://ollama.com/download

# Pull a model
ollama pull deepseek-coder-v2

# Start the server (auto-starts on install)
ollama serve

# Run interactively
ollama run deepseek-coder-v2
```

```bash
# Or use the REST API directly
curl http://localhost:11434/api/chat -d '{
  "model": "deepseek-coder-v2",
  "messages": [{"role": "user", "content": "YOUR_DEVMIND_PROMPT"}],
  "stream": false
}'
```

> Full docs: https://ollama.com/docs
