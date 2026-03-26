---
title: Changelog Generator
category: docs
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Convert raw git commit history or a list of changes into a structured, human-readable CHANGELOG following the Keep a Changelog standard.

## When to Use
- Preparing a release and need to write release notes
- Your git history is full of `fix stuff` commits and you need to clean it for users
- Maintaining a public-facing CHANGELOG.md that non-technical users read

## The Prompt

```
You are a technical writer converting developer commit history into a clean, user-facing changelog.

**Project name:** {{PROJECT_NAME}}
**New version:** {{VERSION}}
**Release date:** {{RELEASE_DATE}}
**Previous version:** {{PREVIOUS_VERSION}}

**Audience:** {{AUDIENCE}}
(Options: "developers" — use technical language; "end users" — plain English, no jargon; "both" — technical with plain-English summaries)

**Raw commits / changes:**
```
{{COMMITS}}
```

**Breaking changes (list explicitly, or write "none"):**
{{BREAKING_CHANGES}}

Using the Keep a Changelog format (https://keepachangelog.com), generate:

1. **Changelog entry** for the new version with sections:
   - `### Added` — New features
   - `### Changed` — Changes to existing functionality
   - `### Deprecated` — Features that will be removed in a future version
   - `### Removed` — Features that were removed
   - `### Fixed` — Bug fixes
   - `### Security` — Security fixes

Rules:
- Group related commits into one bullet point
- Write bullets from the user's perspective ("You can now X" not "We added X")
- For breaking changes, add a `> ⚠️ BREAKING` callout before the entry
- Remove internal/chore commits (bump version, update lockfile, fix typo in test)
- Each bullet should be one line — concise and scannable
- Link to PR numbers if provided

2. **GitHub Release summary** — A 2-3 sentence summary suitable for a GitHub Release description
```

## Example Input

| Variable | Value |
|---|---|
| `{{PROJECT_NAME}}` | `SwiftQueue` |
| `{{VERSION}}` | `2.0.0` |
| `{{RELEASE_DATE}}` | `2025-09-15` |
| `{{PREVIOUS_VERSION}}` | `1.4.2` |
| `{{AUDIENCE}}` | `developers` |
| `{{COMMITS}}` | (see raw commits below) |
| `{{BREAKING_CHANGES}}` | `Queue constructor signature changed — options object is now required` |

**Raw commits:**
```
abc1234 feat: add priority queue support (#142)
def5678 feat: add cron job scheduling with node-cron (#138)
ghi9012 fix: prevent duplicate job IDs on concurrent enqueue (#145)
jkl3456 fix: worker crash when Redis connection drops (#140)
mno7890 chore: bump dependencies
pqr1234 docs: update README examples
stu5678 BREAKING: refactor Queue constructor to require options object (#135)
vwx9012 feat: add job.progress() method for real-time progress updates (#143)
yza3456 security: sanitize job data to prevent prototype pollution (#147)
```

## Example Output

```markdown
## [2.0.0] — 2025-09-15

> ⚠️ **BREAKING CHANGE**: The `Queue` constructor now requires an options object.
> Migrate: `new Queue('name', connectionString)` → `new Queue('name', { redis: connectionString })`

### Added
- **Priority queues** — Assign a `priority` (1-10) to any job; higher-priority jobs are processed first (#142)
- **Cron scheduling** — Schedule recurring jobs using standard cron syntax: `queue.addCron('0 9 * * 1', myJob)` (#138)
- **Job progress tracking** — Call `job.progress(50)` inside your worker to report real-time completion percentage (#143)

### Fixed
- Duplicate job IDs no longer occur when multiple workers enqueue simultaneously (#145)
- Workers now automatically reconnect instead of crashing when Redis drops a connection (#140)

### Security
- Job payload data is now sanitized to prevent prototype pollution attacks (#147)

---

**GitHub Release Summary**

SwiftQueue 2.0 brings priority queues, cron job scheduling, and real-time progress
tracking to your background jobs. This is a major release with one breaking change to
the Queue constructor — see the migration guide above. It also includes a security fix
for prototype pollution that all users should upgrade to immediately.
```

## Tips
- Paste commits in `git log --oneline format` — run `git log v1.4.2..HEAD --oneline` to get exactly the commits for this release
- Always fill in `{{BREAKING_CHANGES}}` explicitly — the AI may misidentify breaking vs. non-breaking from commit messages alone
- For `{{AUDIENCE}}: "end users"`, the AI will strip all technical jargon — useful for SaaS app release notes where users don't care about internals
