# TaskBullet Agent plugin

Escalate work a language model cannot finish to a **human TaskBullet virtual assistant**.

This plugin wires Grok Bot / Cursor to the hosted remote MCP:

**https://taskbullet.com/api/agent-mcp**

It does **not** use `npx @taskbullet/mcp-server` (that is laptop-only stdio). Cloud Grok Bots cannot run npx.

## What you get

| Tool | Purpose |
| --- | --- |
| `delegate_task` | Queue a VA task (title, description, priority, optional webhookUrl) |
| `list_tasks` | Recent delegated tasks |
| `get_task` | Status + result for a `taskId` |

Plus skill **escalate-to-va**: when the model hits phone / 2FA / captcha / physical-world / subjective QA / long manual work, call `delegate_task`. Never invent VA results.

## Install (until the marketplace lists it)

1. Create a key at [taskbullet.com/dashboard/settings/api-keys](https://taskbullet.com/dashboard/settings/api-keys).
2. In Grok Bot: **Settings → Plugins → custom MCP** (or **Add connector**).
3. URL: `https://taskbullet.com/api/agent-mcp`
4. Paste the `tb_live_…` key into connector auth (`x-api-key` or Bearer token).

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

Hours are 402-gated. Unprovisioned accounts return 409 until onboarding is complete. Completions come back on `get_task` and via signed webhooks if you passed `webhookUrl`.

## Marketplace submit

- Cursor: [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) — paste this repo URL
- cursor.directory: [cursor.directory/plugins/new](https://cursor.directory/plugins/new)
- Grok Build: PR to [xai-org/plugin-marketplace](https://github.com/xai-org/plugin-marketplace) pinning this repo SHA

Until those listings are approved, SearchPlugins will not find TaskBullet. Custom URL + key still works.
