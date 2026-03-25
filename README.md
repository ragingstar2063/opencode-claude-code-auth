# opencode-claude-code-auth

[![npm version](https://img.shields.io/npm/v/opencode-claude-code-auth.svg)](https://www.npmjs.com/package/opencode-claude-code-auth)
[![npm downloads](https://img.shields.io/npm/dw/opencode-claude-code-auth.svg)](https://www.npmjs.com/package/opencode-claude-code-auth)
[![publish npm](https://github.com/ragingstar2063/opencode-claude-code-auth/actions/workflows/publish.yml/badge.svg)](https://github.com/ragingstar2063/opencode-claude-code-auth/actions/workflows/publish.yml)
[![release](https://img.shields.io/github/v/release/ragingstar2063/opencode-claude-code-auth?display_name=tag)](https://github.com/ragingstar2063/opencode-claude-code-auth/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Bring back Anthropic auth to OpenCode. Pick your path — Claude Code session, browser OAuth, or plain API key.

## Why this exists

OpenCode `1.3.x` [removed built-in Anthropic auth](https://github.com/anomalyco/opencode/pull/18186). This plugin restores it as a standalone package with two login options and a config helper that sets up the Anthropic provider for you.

## Pick your login path

| | Claude Code session | Browser sign-in | 
|---|---|---|
| **What happens** | Reads your local Claude Code credentials | Opens Anthropic OAuth in your browser | 
| **Needs Claude Code** | Yes | No | 
| **Token refresh** | Triggers the `claude` CLI when tokens get stale | Uses Anthropic OAuth refresh tokens | 
| **Best for** | Anyone already using Claude Code | Anyone without Claude Code installed |

> Both paths are offered during `opencode auth login -p anthropic`. You choose once and the plugin remembers.

---

## Get started

<details open>
<summary><b>Humans</b></summary>

**Quick path — let an LLM handle it**

Paste this into any coding agent:

```
Install the opencode-claude-code-auth plugin and configure Anthropic models in this project's opencode.json by following: https://raw.githubusercontent.com/ragingstar2063/opencode-claude-code-auth/main/README.md
```

**Do it yourself**

1. Add the plugin:

   ```json
   {
     "plugin": ["opencode-claude-code-auth"]
   }
   ```

   > OpenCode installs npm plugins automatically. No `npm install` needed.

2. Run `opencode auth login -p anthropic` and pick a login path.

3. If OpenCode asks whether to configure Anthropic models, say yes.

4. Try it:

   ```bash
   opencode run "Hello" -m anthropic/claude-sonnet-4-5
   ```

</details>

<details>
<summary><b>LLM agents</b></summary>

1. Add `"opencode-claude-code-auth"` to the `plugin` array in `opencode.json`
2. Run `npx opencode-claude-code-auth configure -y`
3. Run `opencode auth login -p anthropic` and pick a method
4. Verify: `opencode run "Hello" -m anthropic/claude-haiku-4-5`

</details>

---

## What it actually does

The plugin hooks into OpenCode's provider system and does three things:

**1. Intercepts Anthropic requests.** It registers an `auth.loader` with a custom `fetch` that takes over all traffic to the Anthropic API. Every outgoing request gets rewritten with the right auth headers, beta flags, tool name format, and system prompt identity before it leaves. Every response stream gets the tool names translated back.

**2. Manages tokens.** Depending on which login path you chose:
- *Claude Code session* — reads tokens from the macOS Keychain (or `~/.claude/.credentials.json` on Linux/Windows), refreshes them by spawning `claude` with stdin closed when they're about to expire, and re-reads the credentials file afterward regardless of exit code.
- *Browser sign-in* — runs a PKCE OAuth flow against Anthropic's authorize endpoint, exchanges the code for tokens via `application/x-www-form-urlencoded` POST, and refreshes them via the token endpoint when they expire.
- *API key* — the plugin steps aside and lets OpenCode handle it natively.

Both OAuth paths sync tokens to OpenCode's `auth.json` via `client.auth.set`. On 401/403 responses, the plugin forces a refresh and retries once.

**3. Patches your config.** On clean OpenCode installs, Anthropic models don't appear until the provider is defined. The plugin can add the Anthropic provider block to `opencode.json` — either through a CLI prompt during login, or via `npx opencode-claude-code-auth configure`.

---

## Where credentials come from

**Claude Code session path:**

| Priority | Source | Platform |
|----------|--------|----------|
| 1 | macOS Keychain (`Claude Code-credentials`) | macOS |
| 2 | `~/.claude/.credentials.json` | All |

**Browser sign-in path:** tokens are stored in OpenCode's auth storage after the initial OAuth exchange.

**API key path:** managed by OpenCode directly.

---

## Models

Any model your Anthropic subscription supports will work. The default config adds:

| Model | ID |
|-------|----|
| Claude Sonnet 4.5 | `claude-sonnet-4-5` |
| Claude Haiku 4.5 | `claude-haiku-4-5` |

Add more by editing the `provider.anthropic.models` block in `opencode.json`.

---

## The config helper

OpenCode `1.3.x` needs explicit Anthropic provider config. The plugin handles this two ways:

**During login** — if Anthropic isn't configured, the CLI asks:

| Option | What happens |
|--------|-------------|
| `Update this project` | Adds provider + default models to `opencode.json` |
| `Skip for now` | Does nothing. Run the CLI command later. |

**From the command line:**

```bash
npx opencode-claude-code-auth configure       # interactive
npx opencode-claude-code-auth configure -y     # skip confirmation
npx opencode-claude-code-auth configure --global  # global config
npx opencode-claude-code-auth doctor           # check what's configured
```

<details>
<summary><b>Full config block (copy-paste)</b></summary>

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["opencode-claude-code-auth"],
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",
  "provider": {
    "anthropic": {
      "npm": "@ai-sdk/anthropic",
      "name": "Anthropic",
      "models": {
        "claude-sonnet-4-5": { "name": "Claude Sonnet 4.5" },
        "claude-haiku-4-5": { "name": "Claude Haiku 4.5" }
      }
    }
  }
}
```

</details>

---

## Request rewriting

The Anthropic OAuth API expects requests shaped differently from standard API-key requests. The plugin handles this transparently:

| What | How |
|------|-----|
| Auth | `Authorization: Bearer <token>`, removes `x-api-key` |
| Beta flags | `anthropic-beta: oauth-2025-04-20,interleaved-thinking-2025-05-14` |
| User-Agent | `claude-cli/2.1.2 (external, cli)` |
| Tool names | Adds `mcp_` prefix on outgoing, strips it on incoming |
| System prompt | Prepends Claude Code identity, rewrites "OpenCode" → "Claude Code" |
| URL | Adds `?beta=true` to `/v1/messages` |
| Model costs | Set to zero (subscription auth, not API credits) |
| Responses | SSE stream rewriting for tool name translation |

---

## Common problems

> **Quick fix**: Run `npx opencode-claude-code-auth configure` then `opencode auth login -p anthropic`.

| Problem | Fix |
|---------|-----|
| Models don't show up | `npx opencode-claude-code-auth configure` |
| Claude Code not installed | Install from [claude.ai/download](https://claude.ai/download), or pick Browser sign-in |
| No Claude session found | `claude auth login --claudeai` |
| Session expired after refresh | `claude auth login --claudeai` |
| Browser code rejected | Start a fresh `opencode auth login -p anthropic` |
| JSON parse error in config | Fix the syntax, then rerun `configure` |

Full guide: [docs/troubleshooting.md](docs/troubleshooting.md)

---

## Overrides

Defaults work for most setups. If Anthropic changes something before an update ships, set an env var:

<details>
<summary><b>Environment variables</b></summary>

| Variable | Default | Purpose |
|----------|---------|---------|
| `CLAUDE_CODE_AUTH_CREDENTIALS_PATH` | `~/.claude/.credentials.json` | Claude credential file |
| `CLAUDE_CODE_AUTH_KEYCHAIN_SERVICE` | `Claude Code-credentials` | macOS Keychain service |
| `CLAUDE_CODE_AUTH_USER_AGENT` | `claude-cli/2.1.2 (external, cli)` | User-Agent header |
| `CLAUDE_CODE_AUTH_BETAS` | `oauth-2025-04-20,...` | Beta flags |
| `CLAUDE_CODE_AUTH_REFRESH_MODEL` | `claude-haiku-4-5` | Model for refresh trigger |
| `CLAUDE_CODE_AUTH_REFRESH_TIMEOUT_MS` | `60000` | Refresh command timeout |
| `CLAUDE_CODE_AUTH_REFRESH_SKEW_MS` | `60000` | How early to consider tokens stale |
| `ANTHROPIC_CLIENT_ID` | *(built-in)* | OAuth client ID |
| `ANTHROPIC_AUTH_URL` | `https://claude.ai/oauth/authorize` | OAuth authorize URL |
| `ANTHROPIC_TOKEN_URL` | `https://console.anthropic.com/v1/oauth/token` | OAuth token URL |
| `ANTHROPIC_REDIRECT_URI` | *(built-in)* | OAuth redirect URI |
| `ANTHROPIC_SCOPE` | *(built-in)* | OAuth scope |

</details>

---

## Files the plugin touches

| File | What | Who writes it |
|------|------|---------------|
| `./opencode.json` | Anthropic provider config | `configure` command or login prompt (with your consent) |
| `~/.local/share/opencode/auth.json` | OAuth tokens | Plugin via `client.auth.set` |
| `~/.config/opencode-claude-code-auth/state.json` | Which login path is active | Plugin |
| `~/.claude/.credentials.json` | Claude Code credentials | Claude Code (read-only by plugin) |

---

## Deeper docs

- [Setup](docs/setup.md) — all install and configure options
- [Auth Methods](docs/auth.md) — how each path works, refresh behavior, decision guide
- [Troubleshooting](docs/troubleshooting.md) — every error and its fix
- [Architecture](docs/architecture.md) — module map, request flow, design constraints

---

## Credits

- [opencode-anthropic-auth](https://www.npmjs.com/package/opencode-anthropic-auth) — original built-in Anthropic auth (deprecated)

## License

MIT. See [LICENSE](LICENSE).

<details>
<summary><b>Disclaimer</b></summary>

This plugin uses Claude Code's OAuth credentials to authenticate with Anthropic's API. Anthropic's Terms of Service state that Claude Pro/Max subscription tokens should only be used with official Anthropic clients. This plugin exists as a community workaround and may stop working if Anthropic changes their OAuth infrastructure.

By using this plugin, you acknowledge:

- **Unofficial** — not endorsed by Anthropic
- **No guarantees** — APIs may change without notice
- **Your risk** — you assume all legal, financial, and technical consequences

Not affiliated with Anthropic. "Claude", "Claude Code", and "Anthropic" are trademarks of Anthropic PBC.

</details>
