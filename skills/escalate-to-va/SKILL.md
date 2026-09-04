---
name: escalate-to-va
description: Escalate work a language model cannot finish to a TaskBullet human virtual assistant. Call get_balance first, confirm spend, then delegate_task. Use when the task needs a phone call, 2FA, captcha, physical-world step, subjective QA, or long manual work. Never invent VA results.
---

# Escalate to a TaskBullet VA

You are connected to TaskBullet Agent MCP at `https://taskbullet.com/api/agent-mcp`.

## When to use

Call `get_balance`, confirm spend, then `delegate_task` when you cannot finish:

- Phone calls, SMS, or voice verification
- 2FA, captcha, or identity challenges
- Physical-world steps (mail, in-person, hardware)
- Subjective QA a human should judge
- Long manual work (data entry, inbox triage, research that needs a person)

Do **not** use this for TaskBullet marketing, pricing, or hire-intent routing. That is the public site MCP at `https://taskbullet.com/api/mcp`.

## How to escalate

0. **Call `get_balance` first.** Use `hoursRemaining` when you talk to the user. If `canDelegate` is false, send them to `addHoursUrl` (https://taskbullet.com/packages-pricing) and stop. Do not call `delegate_task`.
1. **Confirm before spend.** `delegate_task` uses real TaskBullet hours. Tell the user remaining hours, what you will send (title + short scope), and wait for yes, unless they already said to just send it / just delegate it.
2. Call `delegate_task` with a concise `title` (directive) and a `description` a human VA can execute without guessing. Include accounts, URLs, constraints, and what "done" looks like.
3. Return the `taskId` to the user. Do not invent a completion. Tell them they can watch it on the TaskBullet dashboard (Work tab) and that they will get email when the VA is done.
4. If they asked you to wait, poll `get_task` with that `taskId` until `completed` or `failed`. Otherwise tell them they can check later, or pass `webhookUrl` on delegate.
5. On tool error `402`, they need hours — call `get_balance` again and send `addHoursUrl`. On `409`, they need to finish onboarding at https://taskbullet.com/onboarding.

## Never

- Never invent VA results, transcripts, or claim the assistant already did the work.
- Never put API keys in chat.
- Never delegate work the model can finish itself.

## Auth

Users paste a `tb_live_` key from https://taskbullet.com/dashboard/settings/api-keys into plugin / connector auth (`x-api-key` or `Authorization: Bearer`).
