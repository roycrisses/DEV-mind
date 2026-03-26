# Gemini (Google) — Provider Guide

## Overview

Google's **Gemini** family — led by **Gemini 1.5 Pro** and **Gemini 2.0 Flash** — offers the largest available context window in the industry (up to **1 million tokens**), making it uniquely suited for tasks that require ingesting entire codebases, large documentation sets, or long git histories in a single prompt. Gemini 2.0 Flash is also among the fastest frontier models, making it excellent for high-volume, cost-sensitive workflows.

---

## Strengths for Developers

| Strength | Details |
|---|---|
| **1M token context** | Paste your entire repository — something no other provider can match |
| **Multimodal natively** | Can analyze screenshots, architecture diagrams, and UI mockups alongside code |
| **Speed (Flash models)** | Gemini 2.0 Flash is one of the fastest frontier models available |
| **Cost efficiency** | Some of the most competitive per-token pricing among frontier models |
| **Google ecosystem** | Native integration with Google Cloud, Vertex AI, Firebase, and Android Studio |
| **Deep Search grounding** | Can ground responses in real-time web search — useful for "what's the best library for X in 2025?" |

---

## Quirks to Know

- **Verbosity on code tasks** — Gemini can over-explain. Add `"Return only the code. No explanation unless I ask."` when you just want the output.
- **Hallucination on niche APIs** — For obscure libraries or very new frameworks, Gemini may confidently generate plausible-but-wrong API calls. Verify against docs, especially for anything post-2024.
- **Safety blocks on security prompts** — Gemini's safety filters can block legitimate security research prompts (pen testing, vulnerability analysis). Reframe as: `"I'm a security engineer auditing this code for our own application."` Using Vertex AI instead of the public API gives more control over safety settings.
- **Flash vs Pro tradeoffs** — Flash is 5–10x faster and cheaper but less accurate on complex reasoning tasks. Use Flash for `commit-message.md` and `changelog.md`; use Pro for `review.md` and `infra.md`.
- **System instructions in Gemini API** — Gemini uses `system_instruction` at the top level of the request, not as a message in the array. This is different from OpenAI's format.

---

## How to Adjust DevMind Prompts for Gemini

### 1. Exploit the 1M context window for codebase-wide analysis

For the `review.md` and `explain.md` prompts, Gemini is the only model where you can paste an entire multi-file module — or even a whole project — and get a review that understands cross-file dependencies:

```
Here is the entire authentication module (7 files):

[File 1: auth.controller.ts]
{{FILE_CONTENTS_1}}

[File 2: auth.service.ts]
{{FILE_CONTENTS_2}}

... (continue for all files)

Review this as a cohesive system, not file by file.
```

### 2. Use multimodal input for UI and architecture

Paste screenshots along with your prompt in Gemini 1.5 Pro or 2.0:

For `review.md`: attach a screenshot of the UI alongside the React component code — Gemini can review whether the code actually produces what's in the screenshot.

For `infra.md`: give Gemini a photo of your architecture whiteboard — it can interpret the diagram and generate the corresponding Terraform.

### 3. Use system_instruction for persistent role framing

```python
model = genai.GenerativeModel(
    model_name='gemini-1.5-pro',
    system_instruction='You are a senior DevOps engineer. Be direct and specific. Skip disclaimers.'
)
```

### 4. Ground in Search for library recommendations

For prompts that involve technology choice (e.g., `infra.md`'s "why this service over alternatives"), enable Google Search grounding to get up-to-date recommendations:

```python
tools = [genai.Tool(google_search_retrieval=genai.GoogleSearchRetrieval())]
```

---

## Provider-Specific Tips

**Tip 1 — Use Gemini for changelog generation from long git history**
The `changelog.md` prompt shines with Gemini's 1M context. Instead of limiting to the last 50 commits, paste your entire year's git log — Gemini handles it without truncation:
```bash
git log --oneline --since="1 year ago" | pbcopy
# Paste into changelog.md prompt — Gemini will organize it all
```

**Tip 2 — NotebookLM for documentation projects**
For the `api-docs.md` and `readme.md` prompts, Google's NotebookLM (powered by Gemini) allows you to upload your source code as a "source" and ask iterative questions — perfect for large documentation projects.

**Tip 3 — Use Vertex AI for production / no safety restrictions**
For `security audit` type prompts that may hit Gemini's safety filters, deploy through **Google Vertex AI** instead of the public API. Vertex AI gives you safety filter controls and guaranteed SLAs — essential for enterprise use.

```python
import vertexai
from vertexai.generative_models import GenerativeModel

vertexai.init(project="your-gcp-project", location="us-central1")
model = GenerativeModel("gemini-1.5-pro")
```

---

## Comparison vs Other Providers

| Capability | Gemini 1.5 Pro | GPT-4o | Claude 3.5 Sonnet | Ollama |
|---|---|---|---|---|
| **Context window** | ⭐⭐⭐⭐⭐ (1M) | ⭐⭐⭐⭐ (128K) | ⭐⭐⭐⭐⭐ (200K) | ⭐⭐ (8K–32K) |
| **Speed** | ⭐⭐⭐⭐ (Flash: ⭐⭐⭐⭐⭐) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Varies |
| **Cost** | ✅ Very low | 💰 Medium | 💰 Medium | ✅ Free |
| **Multimodal** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ❌ (most models) |
| **Code quality** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **GCP integration** | ✅ Native | ❌ | ❌ | ❌ |

---

## Recommended Models (as of 2025)

| Use Case | Recommended Model |
|---|---|
| Large codebase analysis / review | `gemini-1.5-pro` |
| Fast, cost-sensitive tasks (commit messages, changelogs) | `gemini-2.0-flash` |
| Multimodal (code + UI screenshots) | `gemini-1.5-pro` or `gemini-2.0-pro` |
| Google Cloud / Firebase projects | `gemini-1.5-pro` via Vertex AI |
| Cutting-edge reasoning | `gemini-2.0-pro` |

---

## Quick API Setup

```bash
pip install google-generativeai
export GOOGLE_API_KEY=AIza...
```

```python
import google.generativeai as genai

genai.configure(api_key=os.environ['GOOGLE_API_KEY'])
model = genai.GenerativeModel(
    model_name='gemini-1.5-pro',
    system_instruction='You are a senior software engineer. Be direct and specific.'
)

response = model.generate_content('YOUR_DEVMIND_PROMPT_HERE')
print(response.text)
```

```bash
# Or via REST
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent?key=$GOOGLE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"contents": [{"parts": [{"text": "YOUR_DEVMIND_PROMPT"}]}]}'
```

> Full docs: https://ai.google.dev/docs
