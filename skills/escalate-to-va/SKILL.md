---
name: escalate-to-va
description: Escalate work a language model cannot finish to a TaskBullet human virtual assistant. Call get_balance first, confirm spend, then delegate_task. Use for 2FA, captcha, physical-world steps, subjective QA, or long manual work. Outbound phone calls use place_call. Never invent VA results.
---

# Escalate to a TaskBullet VA

You are connected to TaskBullet Agent MCP at `https://taskbullet.com/api/agent-mcp`.

## Tools

The hosted server exposes these tools:

| Tool | Use |
| --- | --- |
| `get_balance` | Remaining hours, `capRoom`, `canDelegate`, and the calls wallet (`balance_cents`, `holds_cents`, `wallet_cents`). Call this before proposing spend. Does not create a task. |
| `delegate_task` | File a VA brief. Requires `title` and `idempotencyKey`. |
| `list_tasks` | Recent tasks. `status` is `pending`, `in_progress`, `completed`, `failed`, `cancelled`, or `all`. |
| `get_task` | Status, title, and result for one `taskId`. Stop on `completed`, `failed`, or `cancelled`. |
| `confirm_task` | Accept a delivery that is `awaitingCustomerConfirm`. Settles tracked hours. Requires `idempotencyKey`. |
| `cancel_task` | Customer never mind. Releases the hold and bills only tracked time. Requires `idempotencyKey`. |
| `reopen_task` | Send a delivery back with a `reason`. Does not settle hours. Requires `idempotencyKey`. |
| `place_call` | One human phone call. Use skill `place-human-call`, not this one. |
| `validate_call` | Check a call body. Places no hold. |
| `get_call` | One call, including `needs_info` or the final result. |
| `get_call_events` | Per-step log for one call, oldest first. |
| `update_call` | Fill `missing_fields` while a call is `needs_info`. |
| `cancel_call` | Cancel a call. A confirm before any dial refunds the hold. |
| `get_transcript` | VA wrap-up summary. Not a recording. |

## When to use

Call `get_balance`, confirm spend, then `delegate_task` when you cannot finish:

- 2FA, captcha, or identity challenges
- SMS or other verification that is not an outbound business call
- Physical-world steps (mail, in-person, hardware)
- Subjective QA a human should judge
- Long manual work (data entry, inbox triage, research that needs a person)

Outbound US or Canada phone calls use `place_call` (skill `place-human-call`), not `delegate_task`.

Do **not** use this for TaskBullet marketing, pricing, or hire-intent routing. That is the public site MCP at `https://taskbullet.com/api/mcp`.

Named prospect, lead, or cold lists: send the human to https://taskbullet.com/build-a-list (pay once, CSV). Do not scrape contacts and do not spend hours forming the file. After the file exists, calling through it is last-mile `delegate_task`. An existing list skips the wizard.

## How to escalate

0. **Call `get_balance` first.** Use `hoursRemaining` when you talk to the user. If `canDelegate` is false and spendable hours are not above zero, send them to `addHoursUrl` (https://taskbullet.com/packages-pricing) and stop. Do not call `delegate_task`. If `canDelegate` is false while hours remain, lower `maxHours` to `capRoom` or wait. That is not a buy-hours signal. `addHoursUrl` is the packages page, not a new trial.
1. **Confirm before spend.** `delegate_task` uses real TaskBullet hours (15-minute rounding). Tell the user remaining hours, what you will send (title + short scope), and the `maxHours` cap (default 0.5, maximum 8). Wait for yes, unless they already said to just send it / just delegate it. Actual tracked time is billed, not the cap.
2. Call `delegate_task` with a concise `title` (directive, max 5000 characters) and a `description` a human VA can execute without guessing (max 8000). Include accounts, URLs, constraints, and what "done" looks like. Pass `idempotencyKey` (required). The same value is the REST `Idempotency-Key`. A repeat of the same key returns the original task. Optional: `priority` (`low`, `normal`, `high`, `urgent`), `dueDate`, `sourceAgent`, `webhookUrl` (public https), `maxHours`.
3. Return the `taskId` to the user. Status `awaiting_customer_start` means the job is **not** started until the customer clicks Start on Work. Do not say the VA has started. Do not invent a completion. Tell them they can watch it on the TaskBullet dashboard (Work tab) and that they will get email when the VA is done.
4. If they asked you to wait, poll `get_task` with that `taskId` until `completed`, `failed`, or `cancelled`. Otherwise tell them they can check later, or pass `webhookUrl` on delegate. A missing Basecamp connection queues the task. It is not a 409.
5. When `get_task` shows `awaitingCustomerConfirm`, ask the human, then `confirm_task` (settle tracked hours), `cancel_task` (release the hold, bill only tracked time), or `reopen_task` (send it back; `reason` required, at least a few words). Do not confirm, cancel, or reopen without the human. Each of those calls requires its own `idempotencyKey`. If the human does not respond, an agent-created task confirms itself after 3 days.
6. Customer cancel is `cancelled`, not `pending` or `in_progress`, and it does not send the completion webhook. Stop.
7. On tool error `402`, spendable hours are not above zero. Call `get_balance` again and send `addHoursUrl`. On `422`, the cap does not fit open caps: hours remain, lower `maxHours` to `capRoom` or wait. On `503`, hours were unavailable. That is not a buy-hours error. On `409` from confirm, cancel, or reopen, the task is not waiting, already closed, or the idempotency key conflicts.

## Never

- Never invent VA results, transcripts, or claim the assistant already did the work.
- Never put passwords, OTP codes, recovery keys, cookies, or API keys in `delegate_task` fields or in chat.
- Never delegate work the model can finish itself.
- Never use `delegate_task` for an outbound US or Canada phone call. Use `place_call`.

## Auth

Users paste a `tb_live_` key from https://taskbullet.com/dashboard/settings/api-keys into plugin / connector auth (`x-api-key` or `Authorization: Bearer`). API key only — not OAuth.
