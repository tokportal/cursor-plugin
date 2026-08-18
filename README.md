<p align="center">
  <img src="assets/logo.png" alt="TokPortal" width="96" />
</p>

# TokPortal plugin for Cursor

> TokPortal is the managed social infrastructure API: real TikTok, Instagram and YouTube accounts created, warmed and operated by human account managers in 16+ countries — exposed as a REST API and an MCP server. No OAuth per account, no 25-posts/day cap, no app review.

This plugin gives Cursor everything it needs to operate TokPortal:

- **MCP server** (`mcp.json` / `.mcp.json`) — the remote TokPortal MCP server `https://app.tokportal.com/api/ext/mcp` (Streamable HTTP, OAuth 2.1 sign-in on first use; also accepts `Authorization: Bearer sk_...`). 91 `tokportal_*` tools: bundles, videos, uploads, warming, analytics, bans, webhooks, comments.
- **Rules** (`rules/tokportal-api.mdc`) — auth, credits, bundle lifecycle, the 3-videos/day/bundle cap, the 428 confirmation policy for secret-returning tools, and "prefer read-only tools / never paste keys" guidance.
- **Skills** (`skills/*/SKILL.md`) — five step-by-step playbooks: `tokportal-setup`, `tokportal-launch-network`, `tokportal-post-videos-at-scale`, `tokportal-warm-accounts`, `tokportal-monitor-and-bans` (same content as [tokportal/skills](https://github.com/tokportal/skills)).

## Install

### From the Cursor marketplace

Cursor → **Settings → Plugins** (or the Marketplace tab) → search **TokPortal** → Install. Cursor opens the TokPortal OAuth sign-in the first time a `tokportal_*` tool is used; choose **Full access** (read + write) or **Read-only** (never spends credits).

### Manual: MCP only

Click **[Add to Cursor](cursor://anysphere.cursor-deeplink/mcp/install?name=tokportal&config=eyJ1cmwiOiJodHRwczovL2FwcC50b2twb3J0YWwuY29tL2FwaS9leHQvbWNwIn0=)**, or add to `.cursor/mcp.json` (project) / `~/.cursor/mcp.json` (global):

```json
{
  "mcpServers": {
    "tokportal": {
      "url": "https://app.tokportal.com/api/ext/mcp"
    }
  }
}
```

Prefer an API key (from https://app.tokportal.com/developer/api-keys) instead of OAuth? Use the stdio form:

```json
{
  "mcpServers": {
    "tokportal": {
      "command": "npx",
      "args": ["-y", "tokportal-mcp"],
      "env": { "TOKPORTAL_API_KEY": "sk_..." }
    }
  }
}
```

### Manual: rules and skills

Copy `rules/tokportal-api.mdc` into `.cursor/rules/` and `skills/*` into `.cursor/skills/` of your project.

## Verify

Ask Cursor: *"What is my TokPortal credit balance? Use the TokPortal MCP tools."* A working setup calls `tokportal_get_credit_balance` and answers with your balance.

## Layout

```
.cursor-plugin/plugin.json   # plugin manifest
mcp.json                     # MCP server (Cursor plugin format)
.mcp.json                    # same content (Open Plugins / cursor.directory auto-detect)
rules/tokportal-api.mdc
skills/<name>/SKILL.md
assets/logo.png, assets/logo.svg
```

## Links

- MCP guide: https://developers.tokportal.com/mcp
- Cursor + TikTok via MCP: https://developers.tokportal.com/use-cases/ai-agents/cursor-mcp-tiktok
- MCP server source: https://github.com/tokportal/tokportal-mcp
- Skills: https://github.com/tokportal/skills — Claude Code plugin: https://github.com/tokportal/claude-plugin
- Support: team@tokportal.com

## License

MIT — see [LICENSE](LICENSE).
