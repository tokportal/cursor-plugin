---
name: tokportal-setup
description: Connect the TokPortal MCP server (real TikTok, Instagram and YouTube accounts operated by human account managers, exposed as 91 tokportal_* tools) to Claude Code, Claude.ai, Cursor, VS Code, Codex, Gemini CLI, Windsurf, Cline, Goose, Zed, OpenClaw or any stdio host, obtain an API key, verify the connection with tokportal_get_current_user, and read credit costs and balance before any paid action. Use when the user asks to install, connect, authenticate, test or troubleshoot TokPortal in an AI agent host.
metadata:
  author: TokPortal
  version: "1.0.0"
  homepage: https://developers.tokportal.com/mcp
---

# TokPortal setup

TokPortal is the managed social infrastructure API: real TikTok, Instagram and YouTube accounts created, warmed and operated by human account managers in 16+ countries — exposed as a REST API and an MCP server. No OAuth per account, no 25-posts/day cap, no app review.

Two transports, same 91 `tokportal_*` tools:

| Transport | Endpoint / package | Auth |
| --- | --- | --- |
| Remote (Streamable HTTP) | `https://app.tokportal.com/api/ext/mcp` | OAuth 2.1 browser sign-in **or** `Authorization: Bearer sk_...` / `X-API-Key: sk_...` |
| Local (stdio) | `npx -y tokportal-mcp` (Node 18+) | env `TOKPORTAL_API_KEY=sk_...` |

## Step 1 — API key (only for stdio / bearer setups)

1. Tell the user to open https://app.tokportal.com/developer/api-keys, create a key, and copy it once (keys start with `sk_` and are shown only at creation).
2. Never ask the user to paste the key into chat, code or a committed file. Put it in an env var or the host's secret store. If a key leaks, revoke it in the Developer Portal.
3. OAuth setups need no key: the host redirects to TokPortal, the user picks **Full access** (read + write) or **Read-only** (never spends credits), and TokPortal mints a key named `Claude (MCP connector)` that can be revoked at any time.

## Step 2 — Connect the host

Pick the user's host and give them the exact snippet:

- **Claude Code (remote, OAuth)**: `claude mcp add --transport http tokportal https://app.tokportal.com/api/ext/mcp` then run `/mcp` and authenticate in the browser. Stdio alternative: `claude mcp add tokportal -e TOKPORTAL_API_KEY=sk_... -- npx -y tokportal-mcp`. Plugin alternative: `claude plugin marketplace add tokportal/claude-plugin` then `claude plugin install tokportal@tokportal`.
- **Claude.ai / Claude Desktop / ChatGPT / Perplexity**: Settings → Connectors → add custom connector with URL `https://app.tokportal.com/api/ext/mcp`, authentication OAuth.
- **Cursor**: `.cursor/mcp.json` (project) or `~/.cursor/mcp.json` (global): `{"mcpServers":{"tokportal":{"url":"https://app.tokportal.com/api/ext/mcp"}}}` — or the deeplink `cursor://anysphere.cursor-deeplink/mcp/install?name=tokportal&config=eyJ1cmwiOiJodHRwczovL2FwcC50b2twb3J0YWwuY29tL2FwaS9leHQvbWNwIn0=`. Plugin: https://github.com/tokportal/cursor-plugin.
- **VS Code (Copilot agent mode)**: `code --add-mcp '{"name":"tokportal","type":"http","url":"https://app.tokportal.com/api/ext/mcp"}'`.
- **Codex CLI**: in `~/.codex/config.toml` add `[mcp_servers.tokportal]` with `url = "https://app.tokportal.com/api/ext/mcp"`, then `codex mcp login tokportal` (or `bearer_token_env_var = "TOKPORTAL_API_KEY"`).
- **Gemini CLI**: `gemini mcp add --transport http tokportal https://app.tokportal.com/api/ext/mcp` (add `--header "Authorization: Bearer sk_..."` to skip OAuth).
- **Windsurf**: `~/.codeium/windsurf/mcp_config.json` → `{"mcpServers":{"tokportal":{"serverUrl":"https://app.tokportal.com/api/ext/mcp"}}}`.
- **Cline**: `{"mcpServers":{"tokportal":{"type":"streamableHttp","url":"https://app.tokportal.com/api/ext/mcp","headers":{"X-API-Key":"sk_..."}}}}`.
- **Goose**: `goose configure` → Add Extension → Remote Extension (Streaming HTTP) → the URL above.
- **Zed / JetBrains / any stdio host**: command `npx`, args `-y tokportal-mcp`, env `TOKPORTAL_API_KEY=sk_...`.
- **OpenClaw**: `clawhub install tokportal`, then set `TOKPORTAL_API_KEY` in the skill config.
- **n8n**: MCP Client node → HTTP Streamable → endpoint above → Bearer credential with the key.

Full matrix: https://developers.tokportal.com/mcp. Remote connector walkthrough: https://developers.tokportal.com/mcp/remote.

## Step 3 — Verify

1. Call `tokportal_get_current_user`. Expect the workspace/user profile. A 401 means the OAuth flow was not completed or the key is revoked/malformed (must start with `sk_`).
2. Call `tokportal_get_credit_balance` and report `total_credits` plus anything in `expiring_within_7_days`.
3. Call `tokportal_get_credit_costs` and summarize the live prices (do not quote from memory; standard rates at the time of writing: account setup 32 credits, video slot 2, Advanced Niche Warming 5 per target with a 15-credit minimum, Coverage 25 credits / 30 days after the included first period, credential reveal 150 for post-cutoff accounts). If `contract_bundle_allowance` is present, it overrides the standard rate for qualifying bundles.
4. Optionally call `tokportal_list_platforms` and `tokportal_list_countries` to show what can be ordered today (`data` = countries open for new accounts, `videos_only_countries` = existing-account video work only).

If all three calls succeed the setup is complete. Empty tool list → the host cached an old session: remove and re-add the server or restart the host.

## Rules the agent must follow from now on

- Read-only tools (`tokportal_get_*`, `tokportal_list_*`, `tokportal_export_*`) never spend credits and are safe to call freely.
- Before any `create_*`, `configure_*`, `publish_*`, `add_*`, `rewarm_*`, `upload_*` call, check `tokportal_get_credit_costs` and `tokportal_get_credit_balance`, state the expected charge to the user, and get an explicit go-ahead. Pass an `idempotency_key` for safe retries on these tools.
- `tokportal_reveal_account_credentials` and `tokportal_retrieve_account_verification_code` follow the **428 two-step policy**: first call without `body` → HTTP 428 with `policy_version`, price and consequences; show the full disclosure to a human; only after explicit confirmation call again with `body.acknowledge_support_forfeit: true` and the exact `policy_version`. Never send `idempotency_key` to them; on `CREDENTIAL_REVEAL_QUOTE_CHANGED` refetch and re-confirm.
- Never send `idempotency_key` to secret-returning tools (`create_webhook_endpoint`, `create_analytics_report`, `upload_video`/`upload_image` presigned URLs) — the API rejects it with `IDEMPOTENCY_KEY_NOT_ALLOWED_FOR_SENSITIVE_RESPONSE`.
- Destructive tools (`unpublish_bundle`, `reset_bundle_video`, `unschedule_bundle_video`, `delete_webhook_endpoint`, `delete_comment_task`, `cancel_account_managed_subscription`) require a human confirmation first.
- On `RATE_LIMIT_EXCEEDED` wait `diagnostics.retry_after_seconds` (default 120 req/min per key). On `INSUFFICIENT_CREDITS` point to https://app.tokportal.com/dashboard/credits. Quote `request_id` when escalating to team@tokportal.com.

## Troubleshooting quick table

| Symptom | Fix |
| --- | --- |
| 401 loop / "authentication required" | Finish OAuth in the browser tab; re-add the server; for bearer check `sk_` prefix and revocation. |
| Read-only connector cannot create bundles | Re-authorize with Full access. |
| `npx tokportal-mcp` hangs | First-run download; run `npx -y tokportal-mcp` once in a terminal (Node 18+). |
| Corporate proxy blocks `cursor://` / `vscode:` links | Use the JSON/CLI snippet instead. |
