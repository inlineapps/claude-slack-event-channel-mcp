# claude-slack-event-channel-mcp

A one-way (listen-only) MCP server that forwards Slack events into a Claude Code
[channel](https://modelcontextprotocol.io). It connects to Slack via **Socket Mode**,
filters messages/reactions, and pushes them to Claude as `<channel>` notifications.

## Requirements

- [Bun](https://bun.sh)
- A Slack app with **Socket Mode** enabled and an App-Level Token (`xapp-...`) carrying
  the `connections:write` scope.

## Install

```sh
bun install
```

## Configure

Copy `.env.example` to `.env` and fill in the values:

| Variable                 | Required | Description                                                                 |
| ------------------------ | -------- | --------------------------------------------------------------------------- |
| `SLACK_APP_TOKEN`        | yes      | App-Level Token (`xapp-...`), needs `connections:write`.                     |
| `SLACK_WATCH_CHANNELS`   | no       | Comma-separated channel IDs to forward. Empty = all channels.               |
| `SLACK_ALLOWED_SENDERS`  | no       | Comma-separated user IDs / bot_ids allowed. Empty = allow all.              |

> **Security note:** every forwarded message becomes Claude's input — a prompt injection
> entry point. Leaving `SLACK_ALLOWED_SENDERS` empty is only safe for channels that solely
> trust an alert bot. Always allowlist when listening to human messages.

## Run

```sh
bun run start   # or: bun run dev  (watch mode)
```

The server speaks MCP over stdio, so it is normally launched by Claude Code rather than run
standalone. Register it as an MCP server pointing at `main.ts`.

## Scripts

| Script           | Description                          |
| ---------------- | ------------------------------------ |
| `bun run start`  | Run the server.                      |
| `bun run dev`    | Run with file watching.              |
| `bun run check`  | Biome lint + format (auto-fix).      |
| `bun run lint`   | Biome lint only.                     |
| `bun run format` | Biome format (write).                |
| `bun run typecheck` | TypeScript type check.            |
