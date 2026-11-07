# TaskBullet Agent plugin

Escalate work a language model cannot finish to a **human TaskBullet virtual assistant**, and place **human phone calls** in the US and Canada.

This plugin wires the hosted remote MCP:

**https://taskbullet.com/api/agent-mcp**

It does **not** use `npx @taskbullet/mcp-server` (that is laptop-only stdio). Cloud hosts cannot run npx.

The plugin does not ship a client id, a client secret, or a callback URL. The host registers its own public OAuth client. Connection approval is not permission to spend.

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

- **escalate-to-va** — call `get_balance`, state that task's cost, wait for an explicit yes, then `delegate_task`. Never invent VA results.
- **place-human-call** — check the calls wallet, confirm that call's hold, `validate_call`, then `place_call`. Outbound phone calls do not use `delegate_task`.

## Phone-call price

US and Canada calls are **$2.50 for the first minute plus $1 for each additional minute**. `max_minutes` defaults to 5, so the default hold is **$6.50**. Raise `max_minutes` on that call (up to 60) to hold more. The hold is the full call.

A real dial attempt is charged, including no answer and a bad number. A system failure with no dial attempt is fully refunded. Each call is locked to the price at placement. `place_call` is pay-as-you-go from the calls wallet. There is no call trial. `record_call` must be false or omitted. Recording is not available.

## Connect

Grok Build's documented connection is this plugin's `.mcp.json`: the hosted URL, and no static credential header. Grok registers its own OAuth client and opens the browser. In `/mcps`, authenticate that server with `i`. Tokens stay in the host's credential store. Do not paste a key into chat.

That flow uses the hosted authorization contract from [taskbullet-web PR 288](https://github.com/GriffMurder/taskbullet-web/pull/288). It answers only when `MCP_OAUTH_CONNECT` is `1` or `true`. Any other value, including unset, keeps discovery on `oauth_not_supported`. This plugin does not enable that flag.

When the flag is on and the request does not present a `tb_live_` key, a missing credential is challenged with:

`Bearer realm="taskbullet-agent", resource_metadata="https://taskbullet.com/.well-known/oauth-protected-resource"`

The host then uses the discovered server. This plugin does not choose the redirect URI.

| Item | Value |
| --- | --- |
| Resource | `https://taskbullet.com/api/agent-mcp` |
| Protected resource metadata | `https://taskbullet.com/.well-known/oauth-protected-resource` and `https://taskbullet.com/.well-known/oauth-protected-resource/api/agent-mcp` |
| Authorization server metadata | `https://taskbullet.com/.well-known/oauth-authorization-server` and `https://taskbullet.com/.well-known/oauth-authorization-server/api/agent-mcp` |
| Authorize | `https://taskbullet.com/api/agent-mcp/oauth/authorize` |
| Token | `https://taskbullet.com/api/agent-mcp/oauth/token` |
| Register | `https://taskbullet.com/api/agent-mcp/oauth/register` |
| Revoke | `https://taskbullet.com/api/agent-mcp/oauth/revoke` |
| Scope | `agent` |
| Response type | `code` |
| Grants | `authorization_code`, `refresh_token` |
| PKCE | `S256` |
| Token endpoint auth | `none` (public client, no client secret) |
| Bearer | `Authorization` header |
| Client metadata document | supported |

OpenID Connect discovery is not part of this contract. The consent screen is `https://taskbullet.com/connect/agent-mcp`. It shows the host the authorization code returns to, and it says approval gives that app access to paid task delegation and paid phone calls on the account you select. The app does not receive an API key you can copy. Approval stores only a hash.

That approval is access. It is not a yes for any particular task or call. Each `delegate_task` still needs an explicit yes for that task's cost. Each `place_call` still needs an explicit yes for that call's hold.

Cursor's documented remote connection for a server that supports dynamic registration is the same URL, with no static client id and no client secret. Cursor sends its own redirect when it registers. This plugin does not embed that redirect. A Cursor connect is unproven until it is tried against the hosted server after the flag is on.

Grok's published docs do not include a callback URL or a client-registration payload. Do not invent one. A Grok Bot custom connector still uses the pasted-key path below.

## Manual key

A `tb_live_` key remains accepted as `x-api-key` or `Authorization: Bearer`. Create one at [taskbullet.com/dashboard/settings/api-keys](https://taskbullet.com/dashboard/settings/api-keys). Never paste it into chat.

Use this path when the host authenticates with a static header, and while the OAuth flag is off. A static header or token file skips the host's OAuth discovery. A presented `tb_live_` key keeps the API-key challenge (`ApiKey realm="taskbullet-agent"`).

Grok Build:

```toml
[mcp_servers.taskbullet-agent]
url = "https://taskbullet.com/api/agent-mcp"
headers = { "x-api-key" = "tb_live_YOUR_KEY" }
enabled = true
```

Cursor `mcp.json` can send the same header with `${env:TASKBULLET_API_KEY}`. That file is the user's config, not this plugin's `.mcp.json`.

Grok Bot custom connector: URL `https://taskbullet.com/api/agent-mcp`, and the key in connector auth (`x-api-key` or Bearer). That is the connection those docs describe.

## Results

`delegate_task` returns a `taskId`. Later status checks, confirm, cancel, and reopen use that id. A repeat of the same `idempotencyKey` returns the original task.

Report a VA result only from `get_task` for that `taskId`, or from a signed webhook whose `taskId` is that id. `task.completed` carries `taskId`, `taskTitle`, `status`, and `result`. `task.awaiting_confirmation` carries those fields plus `awaitingCustomerConfirm`, `amNote`, and `deliverableLinks`. A customer cancel does not send `task.completed`. Stop on `get_task` status `cancelled`.

`webhookUrl` must be a public https URL. The signature headers are `x-taskbullet-signature` (hex HMAC-SHA256 of `{unixSeconds}.{rawBody}`, no `sha256=` prefix) and `x-taskbullet-timestamp` (unix seconds). The webhook secret is not shown on the OAuth consent screen. If the bot does not have that secret, it uses `get_task`.

Call results come back on `get_call`, `get_call_events`, and `get_transcript`, or on a signed webhook to `callback_url`. Do not invent an outcome.

Hours are 402-gated when spendable hours are not above zero. A cap that does not fit open caps is 422: hours remain, lower `maxHours` to `capRoom` or wait.

## Docs

- HTML: https://taskbullet.com/docs/agent-api
- Markdown: https://taskbullet.com/docs/agent-api.md
- Discovery: https://taskbullet.com/.well-known/mcp.json (`taskbullet-agent`)
- Hosted authorization contract: https://github.com/GriffMurder/taskbullet-web/pull/288

The published agent-api page still describes API-key auth. OAuth on that host follows PR 288, and it is off until the flag is explicitly on.

## Marketplace submit

- Cursor: [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) — paste this repo URL
- cursor.directory: [cursor.directory/plugins/new](https://cursor.directory/plugins/new)
- Grok Build: PR to [xai-org/plugin-marketplace](https://github.com/xai-org/plugin-marketplace) pinning this repo SHA

This change does not move an existing marketplace pin.
