---
name: send-human-email
description: Ask a TaskBullet human to send one email. Check the calls wallet, state the flat fee, wait for an explicit yes, validate_action, then request_action. The default hold is $5.00. Send stays blocked until the human approves the final text. Never invent a send result.
---

# Ask a human to send one email

You are connected to TaskBullet Agent MCP at `https://taskbullet.com/api/agent-mcp`.

A human-sent email uses `request_action`, not `delegate_task` and not `place_call`. VA task work stays on skill `escalate-to-va`. A phone call stays on skill `place-human-call`.

## Tools

| Tool | Use |
| --- | --- |
| `get_balance` | Calls wallet before you propose the fee. The hold comes from that prepaid wallet, not from VA hours. |
| `validate_action` | Same body as `request_action`, without `idempotencyKey`. Creates nothing and places no hold. |
| `request_action` | Ask a human to send one email. Requires `idempotencyKey`. Places the flat-fee hold. |
| `get_action` | Read one action. Never returns the delegated credential. |
| `update_action` | Fill `needs_info`, answer a follow-up, or pass a fresh token after `action.credential_needed`. |
| `approve_action` | Approve the final text. `consent_attestation` must be true. Send stays blocked until this passes. |
| `cancel_action` | Cancel before send. Releases the hold and revokes the credential. After send the error is `cancel_too_late`. |

## Price

The default hold is **$5.00** (500 cents) from the prepaid wallet. Use the `hold_cents` the tool returns. This is not VA hours and not a phone-call hold.

A `needs_info` result charges nothing. An expired token or a scope that does not match the recipients, subject, and message charges nothing.

## How to send

0. **Call `get_balance` first.** If `balance_cents` cannot cover the hold, send `addFundsUrl` and stop.
1. **Confirm before spend.** Tell the user the recipients, the subject, and the hold. Wait for an explicit yes for that fee. Connection consent or a general "send it" is not cost approval.
2. **Call `validate_action`.** It places no hold. Fix `needs_info` before you request. Do not send `request_action` while validation is not ready.
3. **Call `request_action`** with `idempotencyKey` (required; a repeat returns the original action). Required fields: `action_type` (`email`), `recipients`, `subject`, `goal`, `message`, `deadline` (ISO-8601, at least 60 minutes from now), `consent_attestation: true`, `provider` (`google` or `gmail`), and `credential_scope`. The scope must match `recipients`, `subject`, and `message`. Set `consent_attestation` only after the user agrees to this email. `on_behalf_of` may be omitted when signup stored it.
4. A Gmail access token is optional on the request. When you pass `delegated_credential`, also pass `credential_expires_at`. The token is checked with Google, never returned, and never shown to the VA. Do not put the token in chat. A refresh token is not kept.
5. Return the action id. Poll `get_action`, or pass `callback_url`. Do not invent a send result.
6. When the VA asks for approval, show the human the final text and wait for an explicit yes, then `approve_action` with `consent_attestation: true`. If the result is `action.credential_needed`, pass a fresh token with `update_action`. If the VA edited the draft off the approved scope, pass a new `credential_scope` for the approved text. That does not charge again.
7. `cancel_action` before send releases the hold. After send, `cancel_too_late` means the email already went.

## Never

- Never invent a send result or claim the email already went.
- Never put passwords, OTP codes, recovery keys, cookies, API keys, or the Gmail access token in chat or in the action body except inside `delegated_credential`.
- Never approve the final text without the human.
- Never use `delegate_task` or `place_call` to send this email.
- Never ask the user to paste a `tb_live_` key into chat.
