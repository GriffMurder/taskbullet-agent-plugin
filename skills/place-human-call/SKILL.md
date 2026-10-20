---
name: place-human-call
description: Place one US or Canada phone call with a TaskBullet human. Check get_balance, confirm the hold, validate_call, then place_call. $2.50 for the first minute plus $1 per additional minute; the default 5-minute hold is $6.50. Never invent call results.
---

# Place a human phone call

You are connected to TaskBullet Agent MCP at `https://taskbullet.com/api/agent-mcp`.

Human phone calls use `place_call`, not `delegate_task`. Work that is not this call (2FA, captcha, physical-world steps, subjective QA, long manual work) stays on skill `escalate-to-va`.

## Tools

| Tool | Use |
| --- | --- |
| `get_balance` | Calls wallet before you propose spend. `balance_cents` is spendable after open holds. `holds_cents` is unsettled holds. `wallet_cents` is `balance_cents` plus `holds_cents`. `addFundsUrl` does not create a Stripe session. |
| `validate_call` | Same body as `place_call`, without `idempotencyKey`. Creates nothing and places no hold. |
| `place_call` | Place one call. Requires `idempotencyKey`. Pay-as-you-go. No trial. |
| `get_call` | Read one call, including `needs_info`, `missing_fields`, wrap-up, cancel, refund, or the final result. |
| `get_call_events` | Durable per-step log for one `call_id`, oldest first (`seq`, `kind`, `status`, `outcome`, `release_reason`, `summary`). |
| `update_call` | Fill `missing_fields` while the call is `needs_info`. |
| `cancel_call` | Cancel. Status stays `cancel_pending` until ops confirms. |
| `get_transcript` | VA wrap-up summary (`source` `wrap_up_summary`, `recorded` false). Not a recording. |
| `delegate_task`, `list_tasks`, `get_task`, `confirm_task`, `cancel_task`, `reopen_task` | VA task tools. See skill `escalate-to-va`. Do not use them to place this call. |

## Price

US and Canada only. **$2.50 for the first minute plus $1 for each additional minute.** `max_minutes` defaults to 5, so the default hold is **$6.50**. Raise `max_minutes` on that call to hold more, up to 60. The hold is the full call.

A real dial attempt is charged, including no answer and a bad number. A system failure with no dial attempt is fully refunded. Each call is locked to the price at placement: settlement uses the quote that hold paid for and never charges more than that hold.

## How to place

0. **Call `get_balance` first.** If `balance_cents` cannot cover the hold, send `addFundsUrl` and stop. HTTP 402 `payment_required` means the same thing.
1. **Confirm before spend.** Tell the user the number, the goal, and the hold (`$6.50` at the default 5 minutes, or the amount for the `max_minutes` you will send). Wait for yes, unless they already said to place the call.
2. **Call `validate_call`.** It places no hold. A ready body includes `hold_cents` and whether `recipient_timezone` was inferred. `valid` can stay true when the balance is short; `warnings` then includes `insufficient_balance` with `needed_cents` and `addFundsUrl`. Fix `needs_info` fields before you place. Do not send `place_call` while validation is not ready.
3. **Call `place_call`** with `idempotencyKey` (required; a repeat returns the original call). Required fields: `phone` (US or Canada; stored as E.164), `business_name`, `purpose` (`info`, `scheduling`, `account_servicing`, or `vendor_ops`), `goal` (max 4000 characters), `questions` (1 to 20 items of `{id, ask}`; `id` max 80 characters, `ask` at least 8 and at most 1000), `on_behalf_of`, `deadline` (ISO-8601), `consent_attestation: true`. Set `consent_attestation` only after the user confirms they may place this call. `record_call` must be false or omitted. Sales purposes are rejected and place no hold.
4. The deadline needs 60 minutes of weekday 09:00–17:00 in the recipient's time zone. Set `allow_outside_business_hours: true` only after the customer confirms the call may happen outside that window. Omit `recipient_timezone` only when TaskBullet can infer it from the area code; otherwise the call comes back `needs_info`.
5. Return the `call_id`. Poll `get_call`, or pass `callback_url`. Do not invent an outcome, a transcript, or question answers.
6. On `needs_info`, fill `missing_fields` with `update_call`. On `cancel_call`, a confirm before any dial refunds the hold. If the dial already started, the error is `cancel_too_late` and the call bills normally.
7. Read `get_transcript` when the call is `wrap_up` or `completed`. Before a wrap-up summary exists, the result is `transcript_not_ready`. A call that ended without a dial is "no transcript: call not placed". A cancel after the dial started is `call_cancelled`. Use `get_call_events` for the step log. A queued `place_call` appends `call.placed`. Replaying the same idempotency key does not append those rows again.

## Never

- Never invent call results, transcripts, or claim the person already dialed.
- Never put passwords, OTP codes, recovery keys, cookies, or API keys in the call body or in chat.
- Never set `record_call` to true.
- Never use `delegate_task` to place this phone call.
