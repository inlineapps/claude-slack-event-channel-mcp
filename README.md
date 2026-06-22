# claude-slack-event-channel-mcp

A one-way (listen-only) [Claude Code **channel**](https://code.claude.com/docs/en/channels-reference)
that forwards Slack events into a Claude Code session. It connects to Slack via **Socket Mode**,
filters messages and reactions, and pushes them to Claude as `<channel>` notifications so Claude
can react to things happening in Slack without leaving the terminal.

## What is a channel?

A channel is an [MCP](https://modelcontextprotocol.io) server that Claude Code spawns as a
subprocess and talks to over stdio. Instead of waiting for the user to type, the channel *pushes*
events into the session. One-way channels (like this one) forward alerts, webhooks, or chat
messages for Claude to act on; two-way channels also expose a reply tool so Claude can send
messages back. This server is one-way: Slack → Claude only.

The channel contract is just three things, all implemented in `main.ts`:

1. **Declare the capability** — the `experimental: { 'claude/channel': {} }` key in the
   `McpServer` constructor is what registers the notification listener and makes this an MCP
   server a *channel*.
2. **Emit notifications** — for each Slack event, the server calls
   `mcp.server.notification({ method: 'notifications/claude/channel', params: { content, meta } })`.
3. **Connect over stdio** — `mcp.connect(new StdioServerTransport())`; Claude Code owns the process.

The `instructions` string in the constructor is added to Claude's system prompt so Claude knows
what these events mean and that the channel is one-way.

### How an event reaches Claude

When a Slack event passes the filters, `content` becomes the body of a `<channel>` tag and each
`meta` key becomes a tag attribute (the `source` attribute is filled in automatically from the
server's configured name, `slack`). For example a thread reply arrives in Claude's context as:

```text
<channel source="slack" kind="message" channel="C0123ABC" user="U0456DEF" ts="1718000000.000100" thread_ts="1718000000.000001">
deploy looks stuck, can you check?
</channel>
```

`kind="reaction_added"` / `kind="reaction_removed"` events carry a `reaction` emoji name and the
target `ts` instead.

> **Note on `meta` keys:** keys must be identifiers (letters, digits, underscores only). Keys with
> hyphens or other characters are silently dropped, which is why this server uses `thread_ts`, not
> `thread-ts`.

> **Delivery is fire-and-forget:** notifications are not acknowledged. If the session hasn't loaded
> this server as a channel, or org policy blocks it, events are dropped silently with no error. Events
> that arrive while Claude is busy are queued and delivered together on the next turn.

## Requirements

- [Bun](https://bun.sh)
- Claude Code **v2.1.80 or later** (channels are in [research preview](https://code.claude.com/docs/en/channels)).
  On Team/Enterprise plans an admin must enable channels first.
- A Slack app with **Socket Mode** enabled and an App-Level Token (`xapp-...`) carrying the
  `connections:write` scope. Subscribe the app to the events you want forwarded (`message.channels`,
  `reaction_added`, `reaction_removed`, …) and invite it to the target channels.

## Install

```sh
bun install
```

## Configure

Copy `.env.example` to `.env` and fill in the values (loaded via `dotenv`):

| Variable                | Required | Description                                                    |
| ----------------------- | -------- | -------------------------------------------------------------- |
| `SLACK_APP_TOKEN`       | yes      | App-Level Token (`xapp-...`), needs `connections:write`.       |
| `SLACK_WATCH_CHANNELS`  | no       | Comma-separated channel IDs to forward. Empty = all channels.  |
| `SLACK_ALLOWED_SENDERS` | no       | Comma-separated user IDs / bot_ids allowed. Empty = allow all. |

> **Security — this is a prompt injection vector.** Every forwarded message becomes Claude's input.
> Anyone who can post in a watched channel can put text in front of Claude. Leaving
> `SLACK_ALLOWED_SENDERS` empty is only safe for channels that solely trust an alert bot; always set
> an allowlist when listening to human messages. The server gates on the *sender's* identity
> (`user` / `bot_id`), not the channel, so membership in a watched channel alone is not enough.

## Register with Claude Code

Claude Code launches the server itself — you do not run it standalone. Add it to your MCP config so
Claude Code spawns it as a subprocess. Project-level `.mcp.json` (relative path):

```json
{
  "mcpServers": {
    "slack": { "command": "bun", "args": ["./main.ts"] }
  }
}
```

For user-level config in `~/.claude.json`, use the absolute path to `main.ts` so it resolves from
any project.

### Run during the research preview

Custom channels aren't on the approved allowlist yet, so start Claude Code with the development flag
(this bypasses the allowlist for this one entry after a confirmation prompt; org policy still applies):

```sh
claude --dangerously-load-development-channels server:slack
```

The first time, Claude Code asks for consent to use the new server from `.mcp.json` — select **Use
this MCP server**. A dim notice under the startup banner confirms the channel is registered. Once
running, posting in a watched Slack channel pushes the event straight into the session.

If events don't arrive, run `/mcp` in the session to check the server's status; "Failed to connect"
usually means an import/dependency error — check `~/.claude/debug/<session-id>.txt`.

## Local development

You can run the server directly to verify it connects to Slack (it will throw if `SLACK_APP_TOKEN`
is missing). Note that outside Claude Code there's no channel listener, so notifications go nowhere —
this is only useful for checking the Socket Mode connection and your filters.

```sh
bun run start   # or: bun run dev  (watch mode)
```

## Scripts

| Script              | Description                     |
| ------------------- | ------------------------------- |
| `bun run start`     | Run the server.                 |
| `bun run dev`       | Run with file watching.         |
| `bun run check`     | Biome lint + format (auto-fix). |
| `bun run lint`      | Biome lint only.                |
| `bun run format`    | Biome format (write).           |
| `bun run typecheck` | TypeScript type check.          |

## See also

- [Channels reference](https://code.claude.com/docs/en/channels-reference) — the full channel contract.
- [Channels](https://code.claude.com/docs/en/channels) — installing and enabling channels.
- [Working channel implementations](https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins)
  (Telegram, Discord, iMessage, fakechat) — including two-way reply tools and sender-pairing flows.
