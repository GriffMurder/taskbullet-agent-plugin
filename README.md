# TaskBullet Agent plugin for Grok Build

Escalate work a language model cannot finish to a [TaskBullet](https://taskbullet.com) human virtual assistant. This plugin also places US and Canada phone calls and can ask a human to send one email.

The work runs at TaskBullet. Nothing is installed on the user's machine and no code runs locally: the plugin is one MCP server config and three skills.

## Install

1. Install Grok Build and sign in (see the [Grok Build docs](https://docs.x.ai/build/overview)).
2. Start `grok`, run `/marketplace`, find **taskbullet-agent**, and install it. Or from a terminal:

   ```bash
   grok plugin install taskbullet-agent --trust
   ```

3. Set the plugin variable `TASKBULLET_API_KEY` to a `tb_live_` key from [API keys](https://taskbullet.com/dashboard/settings/api-keys). Grok sends that value as the `x-api-key` header. The key is not in this repository. Do not paste it into chat.
4. The hosted server does not advertise OAuth. Do not start an OAuth flow. Follow OAuth only when the server advertises it.

## What is in the plugin

| Component | Path | Purpose |
|---|---|---|
| MCP server | `.mcp.json` | TaskBullet's hosted server over HTTP |
| Skill | `skills/escalate-to-va/SKILL.md` | When to hand unfinished work to a human VA |
| Skill | `skills/place-human-call/SKILL.md` | When to place one US or Canada phone call |
| Skill | `skills/send-human-email/SKILL.md` | When to ask a human to send one email |

There are no hooks, commands, agents, scripts, local binaries, or postinstall steps.

## Tools

The plugin connects to the hosted server. `tools/list` is the live tool list. These are the tools that server documents:

| Group | Tools |
|---|---|
| Balance | `get_balance` |
| VA tasks | `delegate_task`, `list_tasks`, `get_task`, `confirm_task`, `cancel_task`, `reopen_task` |
| Phone calls | `validate_call`, `place_call`, `get_call`, `get_call_events`, `update_call`, `cancel_call`, `get_transcript` |
| Human email | `validate_action`, `request_action`, `get_action`, `update_action`, `approve_action`, `cancel_action` |

`delegate_task` files a brief and waits for the customer to click Start. It does not dial and it does not send email. A phone call is `place_call`. An email a human sends is `request_action`.

## Network endpoints and credentials

| Endpoint | Why |
|---|---|
| `https://taskbullet.com/api/agent-mcp` | The only endpoint this plugin calls. Every tool call goes here. |

The plugin itself contacts nothing else and sends no telemetry. It does not read `~/.ssh`, `.env`, tokens, or environment variables. The host substitutes the user-configured `TASKBULLET_API_KEY` into the `x-api-key` header.

Credentials required: the user's own TaskBullet API key. Header auth is `x-api-key: tb_live_…`. `Authorization: Bearer tb_live_…` is also accepted by the server. Unauthenticated requests do not run tools.

Paid actions need an explicit yes for that action's quoted cost or stated hold. Connection consent or a general "send it" is not cost approval.

## Links

- [TaskBullet Agent API](https://taskbullet.com/docs/agent-api)
- [API keys](https://taskbullet.com/dashboard/settings/api-keys)

## License

MIT. See [LICENSE](LICENSE).
