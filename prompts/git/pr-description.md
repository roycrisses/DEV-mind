---
title: Pull Request Description Writer
category: git
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Write a clear, structured PR description that gives reviewers everything they need to understand, test, and approve your change quickly — without a Slack thread of follow-up questions.

## When to Use
- Opening a PR and your description is blank or just "see commit messages"
- Submitting a PR to an open-source project with formal review processes
- When the change is complex and you want reviewers to focus on the right things

## The Prompt

```
You are a senior engineer writing a pull request description. Your goal is to make the reviewer's job as easy as possible — they should understand what changed, why, the risks, and how to test it without reading every line of code.

**Repository context:** {{REPO_CONTEXT}}

**PR title:** {{PR_TITLE}}
(Should follow Conventional Commits: "feat(auth): add OAuth2 Google login")

**What changed:**
{{WHAT_CHANGED}}
(High-level summary — bullet points of the main changes)

**Why this change is needed:**
{{WHY}}
(The business or technical reason — link to ticket/issue if possible)

**Type of change:**
{{CHANGE_TYPE}}
(Options: "new feature", "bug fix", "refactor", "performance improvement", "docs only", "breaking change", "dependency update")

**How to test this:**
{{HOW_TO_TEST}}
(The exact steps a reviewer needs to verify this works — be specific)

**Screenshots / recordings (if UI change):**
{{SCREENSHOTS}}
(Describe what to include, or write "N/A — backend only")

**Risks / things to watch for:**
{{RISKS}}
(What could go wrong? What should reviewers scrutinize most? Or write "low risk — no state changes")

**Related issues / tickets:**
{{RELATED_ISSUES}}
(Examples: "Closes #142", "Part of EPIC-89", "none")

Generate a PR description using this template:

## Summary
[What this PR does in 2-3 sentences]

## Why
[The motivation — link to ticket]

## Changes
[Categorized list: Frontend / Backend / Database / Config / etc.]

## How to Test
[Numbered steps — exact, reproducible]

## Screenshots
[If applicable]

## Risks & Review Notes
[What to scrutinize, any known limitations]

## Checklist
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No sensitive data in code
- [ ] Reviewed my own diff
```

## Example Input

| Variable | Value |
|---|---|
| `{{REPO_CONTEXT}}` | `React + Node.js SaaS application — team of 5 engineers` |
| `{{PR_TITLE}}` | `feat(billing): add Stripe subscription management` |
| `{{WHAT_CHANGED}}` | `New /billing page, Stripe Checkout integration, webhook handler for subscription events, user.subscription_status field added to DB` |
| `{{WHY}}` | `We've been manually processing payments via invoice — this automates it and unblocks self-serve signups` |
| `{{CHANGE_TYPE}}` | `new feature` |
| `{{HOW_TO_TEST}}` | `Use Stripe test card 4242 4242 4242 4242, go to /billing, click Subscribe` |
| `{{RISKS}}` | `Webhook signature verification is new — if the STRIPE_WEBHOOK_SECRET env var is missing, all webhooks silently fail` |
| `{{RELATED_ISSUES}}` | `Closes #91, Part of EPIC-12` |

## Example Output

```markdown
## Summary

Adds a complete Stripe subscription billing flow to support self-serve signups.
Users can now subscribe, upgrade, and cancel plans directly from the app without
involving the sales team. This replaces our manual invoice process entirely.

Closes #91 | Part of EPIC-12

---

## Why

Manual invoicing is blocking self-serve growth. Every new customer currently requires
a sales call + manual Stripe invoice. This feature lets users sign up and pay
independently, which is required before we can run paid ads profitably.

---

## Changes

**Frontend**
- New `/billing` page with current plan display and upgrade/cancel actions
- Stripe.js `<CardElement>` integrated in checkout modal
- Subscription status badge added to account settings sidebar

**Backend**
- `POST /api/billing/create-checkout-session` — creates Stripe Checkout session
- `POST /api/webhooks/stripe` — handles `customer.subscription.updated/deleted` events
- Stripe webhook signature verification middleware added

**Database**
- New column: `users.subscription_status` (enum: free | active | cancelled | past_due)
- New column: `users.stripe_customer_id` (nullable string)
- Migration: `20250915_add_subscription_fields.sql`

---

## How to Test

1. Checkout the branch: `git checkout feat/billing`
2. Set `STRIPE_WEBHOOK_SECRET` in `.env` (grab from Stripe Dashboard > Webhooks > staging endpoint)
3. Start local Stripe event forwarding: `stripe listen --forward-to localhost:3000/api/webhooks/stripe`
4. Navigate to `/billing` — should show "Free Plan"
5. Click "Upgrade" — enter test card `4242 4242 4242 4242`, any future date, any CVC
6. Confirm redirect back to `/billing` shows "Pro Plan — Active"
7. Test cancellation: click "Cancel Plan" — should show "Cancels at end of billing period"
8. Test failed payment: use card `4000 0000 0000 9995` — should show "Payment Failed" status

---

## ⚠️ Risks & Review Notes

- **Webhook env var** — If `STRIPE_WEBHOOK_SECRET` is missing or wrong in production, all subscription events silently fail. There's a startup check that logs a warning but does NOT throw. Consider making it fatal — noted in TODO on line 34 of `webhook.middleware.ts`.
- **Idempotency** — Stripe webhooks can duplicate. The handler checks `stripe_event_id` in the DB before processing. Please verify this logic in `billing.service.ts:handleWebhookEvent()`.
- **No refund flow yet** — Out of scope for this PR. Manual refunds via Stripe Dashboard for now.

---

## Checklist
- [x] Unit tests for webhook handler (with Stripe fixture payloads)
- [x] E2E test for happy-path subscription
- [ ] Documentation updated (not needed — internal-only feature)
- [x] No sensitive data in code (Stripe keys in env only)
- [x] Reviewed my own diff
```

## Tips
- The `{{HOW_TO_TEST}}` section is the most important one — reviewers who can't reproduce the feature will either approve blindly or block the PR with questions
- Include the exact test card numbers, environment variable names, and CLI commands — zero ambiguity
- Use `{{RISKS}}` to direct reviewer attention to the parts of the code that _actually_ need scrutiny — this makes reviews 3x faster and more productive
