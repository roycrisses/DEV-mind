---
title: Conventional Commit Message Writer
category: git
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Generate clean, Conventional Commits-compliant commit messages from a git diff or plain-English description of changes — no more `fix stuff` or `changes` commits.

## When to Use
- You have a staged diff and need a commit message that actually describes what changed
- You want to enforce Conventional Commits format across your team
- You're writing a commit for a change that touches multiple files and you need help summarizing it

## The Prompt

```
You are an expert in writing clean, informative git commit messages following the Conventional Commits specification (https://www.conventionalcommits.org).

**Repository context:**
{{REPO_CONTEXT}}
(Examples: "Node.js REST API for a fintech app", "React component library", "open-source Python CLI tool")

**The change I made (choose one format):**

Option A — Paste the git diff:
```diff
{{GIT_DIFF}}
```

Option B — Plain English description:
{{CHANGE_DESCRIPTION}}

**Is this a breaking change?** {{IS_BREAKING}}
(Yes / No)

**Related issue / ticket number (if any):** {{ISSUE_NUMBER}}
(Examples: "#142", "JIRA-891", "none")

Generate:
1. **Commit message** — Following this exact format:
   ```
   <type>(<scope>): <subject>

   <body — optional, only if the why isn't obvious>

   <footer — BREAKING CHANGE or closes #issue>
   ```

   Rules:
   - Type must be one of: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, `revert`
   - Subject: imperative mood ("add X" not "added X"), max 72 chars, no period at end
   - Scope: the module, file, or area affected (e.g., `auth`, `api`, `ui/button`)
   - Body: explain the *why*, not the *what* — the diff shows the what
   - Footer: `BREAKING CHANGE: <description>` if applicable, `Closes #<number>` if applicable

2. **Alternative messages** — 2 shorter variations (single-line only) if the user prefers brevity

3. **Explanation** — Why you chose that type and scope
```

## Example Input

| Variable | Value |
|---|---|
| `{{REPO_CONTEXT}}` | `Node.js Express API for a B2B SaaS app` |
| `{{GIT_DIFF}}` | Diff showing `bcrypt.compare()` replaced with `argon2.verify()`, removal of bcrypt dep, addition of argon2 dep in package.json |
| `{{IS_BREAKING}}` | `No` |
| `{{ISSUE_NUMBER}}` | `#203` |

## Example Output

```
**Commit Message**

refactor(auth): replace bcrypt with argon2 for password hashing

bcrypt's cost factor doesn't scale well on modern hardware and is
vulnerable to GPU-based attacks at low cost factors. Argon2id is the
winner of the Password Hashing Competition and is recommended by
OWASP as the preferred algorithm for new applications.

Closes #203

---

**Alternative Messages (brief)**
- `refactor(auth): migrate password hashing from bcrypt to argon2id`
- `perf(auth): switch to argon2id for stronger, faster password hashing`

**Explanation**
Type `refactor` was chosen because existing behavior (password verification) is unchanged — only the internal algorithm changed. `feat` would be wrong since no new capability is added. Scope `auth` because only the authentication module was touched. The body explains the security reasoning — future readers will understand *why* argon2 was chosen without hunting through Slack history.
```

## Tips
- Paste the actual `git diff --staged` output for the most accurate result — it's more reliable than describing changes in English
- Use `{{IS_BREAKING}}: Yes` when the change affects any public API, CLI behavior, or configuration format — the AI will add the mandatory `BREAKING CHANGE:` footer
- For atomic commits (one logical change per commit), this prompt works best. If your diff is huge and touches 5 unrelated things, split before committing
