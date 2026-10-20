# TaskBullet Agent plugin

Escalate work a language model cannot finish to a **human TaskBullet virtual assistant**, and place **human phone calls** in the US and Canada.

This plugin wires Grok Bot / Cursor to the hosted remote MCP:

**https://taskbullet.com/api/agent-mcp**

It does **not** use `npx @taskbullet/mcp-server` (that is laptop-only stdio). Cloud Grok Bots cannot run npx.

## What you get

| Tool | Purpose |
| --- | --- |
| `get_balance` | Remaining VA hours, cap room, and the calls wallet. Call this before proposing spend. |
| `delegate_task` | Queue a VA task (title, description, priority, dueDate, sourceAgent, optional webhookUrl and maxHours). Requires `idempotencyKey`. |
| `list_tasks` | Recent delegated tasks (title, status, short result). |
| `get_task` | Status and result for a `taskId`. |
| `confirm_task` | Accept a delivery that is waiting on the customer. Settles tracked hours. Ask the human first. |
| `cancel_task` | Customer never mind. Releases the hold and bills only tracked time. Ask the human first. |
| `reopen_task` | Send a delivery back to the account manager with a reason. Does not settle hours. Ask the human first. |
| `place_call` | Place one human phone call (US and Canada). Requires `idempotencyKey`. |
| `validate_call` | Check a call body. Creates nothing and places no hold. |
| `get_call` | Read one call, including `needs_info` or the final result. |
| `get_call_events` | Per-step log for one call, oldest first. |
| `update_call` | Fill missing fields while a call is `needs_info`. |
| `cancel_call` | Cancel a call. A cancel confirmed before any dial refunds the hold. |
| `get_transcript` | VA wrap-up summary for a call. Not a recording. |

Skills:

- **escalate-to-va** — call `get_balance` first, confirm spend, then `delegate_task` for 2FA, captcha, physical-world steps, subjective QA, or long manual work. Never invent VA results.
- **place-human-call** — check the calls wallet, confirm the hold, `validate_call`, then `place_call`. Outbound phone calls do not use `delegate_task`.

## Phone-call price

US and Canada calls are **$2.50 for the first minute plus $1 for each additional minute**. `max_minutes` defaults to 5, so the default hold is **$6.50**. Raise `max_minutes` on that call (up to 60) to hold more. The hold is the full call.

A real dial attempt is charged, including no answer and a bad number. A system failure with no dial attempt is fully refunded. Each call is locked to the price at placement. `place_call` is pay-as-you-go from the calls wallet. There is no call trial. `record_call` must be false or omitted. Recording is not available.

## Install (until the marketplace lists it)

1. Create a key at [taskbullet.com/dashboard/settings/api-keys](https://taskbullet.com/dashboard/settings/api-keys).
2. In Grok Bot: **Settings → Plugins → custom MCP** (or **Add connector**).
3. URL: `https://taskbullet.com/api/agent-mcp`
4. Paste the `tb_live_…` key into connector auth (`x-api-key` or Bearer token). This is **API key only — not OAuth**. If a connect card / `needsAuth` appears, dismiss it; header auth is enough. Plugin variable: `TASKBULLET_API_KEY`.

Cursor / Grok Build `mcp.json`:

```toml
[mcp_servers.taskbullet_agent]
url = "https://taskbullet.com/api/agent-mcp"
headers = { "x-api-key" = "tb_live_YOUR_KEY" }
enabled = true
```

Team marketplace: import this GitHub repo. Set plugin variable `TASKBULLET_API_KEY` under **Plugins → Configure**.

## Auth

- Header `x-api-key: tb_live_…`
- `Authorization: Bearer tb_live_…` also accepted
- Unauthenticated GET returns public metadata (`auth.required: true`). Tools stay locked.

## Docs

- HTML: https://taskbullet.com/docs/agent-api
- Markdown: https://taskbullet.com/docs/agent-api.md
- Discovery: https://taskbullet.com/.well-known/mcp.json (`taskbullet-agent`)

Hours are 402-gated when spendable hours are not above zero. A cap that does not fit open caps is 422: hours remain, lower `maxHours` to `capRoom` or wait. Completions come back on `get_task` and via signed webhooks if you passed `webhookUrl`. Call results come back on `get_call`, `get_call_events`, and `get_transcript`.

## Marketplace submit

- Cursor: [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) — paste this repo URL
- cursor.directory: [cursor.directory/plugins/new](https://cursor.directory/plugins/new)
- Grok Build: PR to [xai-org/plugin-marketplace](https://github.com/xai-org/plugin-marketplace) pinning this repo SHA

Until those listings are approved, SearchPlugins will not find TaskBullet. Custom URL + key still works.
