# Per-tool setup: routing coding agents through a gateway

Exact configuration for pointing each coding agent at a Bifrost gateway with a per-developer virtual key (`sk-bf-...`). Replace `https://bifrost.internal.example.com` with your gateway URL (`http://localhost:8080` for a local trial). Model IDs below are examples; use the models your gateway actually has configured and the virtual key allows.

Source pages: https://docs.getbifrost.ai/cli-agents/overview.md and the per-agent pages linked in each section. Agent CLIs change often; if a flag or menu label does not match, update the agent first, then re-check the linked page.

## Before any agent: check the key and model

```bash
GW=https://bifrost.internal.example.com
VK=sk-bf-your-virtual-key

# Anthropic surface (Claude Code)
curl -sS -w '\n%{http_code}\n' "$GW/anthropic/v1/messages" \
  -H "Authorization: Bearer $VK" -H "anthropic-version: 2023-06-01" -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":16,"messages":[{"role":"user","content":"ping"}]}'

# OpenAI-compatible surface (Codex, Cursor, Copilot, OpenCode)
curl -sS -w '\n%{http_code}\n' "$GW/v1/chat/completions" \
  -H "Authorization: Bearer $VK" -H "content-type: application/json" \
  -d '{"model":"openai/gpt-5","messages":[{"role":"user","content":"ping"}]}'
```

`200` = URL, key and model are fine. `401` = key rejected. `403` = key does not allow that provider/model (a new key with no `provider_configs` denies everything).

## URL shapes at a glance

| Agent | Base URL | Notes |
|---|---|---|
| Claude Code | `<gw>/anthropic` | Anthropic Messages API |
| Codex CLI | `<gw>/openai/v1` | Trailing `/v1` required; Responses API |
| Gemini CLI | `<gw>/genai` | API-key mode only |
| Cursor | `<gw>/cursor` | Must be publicly reachable |
| Copilot app | `<gw>/openai/v1/chat/completions` (OpenAI-compatible) or `<gw>/anthropic` | Full path for the app |
| Copilot CLI | `<gw>/openai/v1` or `<gw>/anthropic` | Env vars |
| Copilot Chat (VS Code) | per-model `url`: `<gw>/openai/v1/chat/completions` | `chatLanguageModels.json` |
| OpenCode | `<gw>/openai` | `@ai-sdk/openai-compatible` |

Any model a coding agent uses must support tool calling and streaming. A model that answers chat can still fail mid-task; validate each model with a real agent task before adding it to an allowlist.

## Option A: Bifrost CLI launcher

Works for Claude Code, Codex CLI, Gemini CLI and OpenCode. It stores the virtual key in the OS keyring and starts the agent with gateway URL, key and model set for that process only, leaving the agent's own config files untouched.

```bash
npx -y @maximhq/bifrost-cli                    # installs to ~/.bifrost/bin/bifrost
bifrost context add production --base-url https://bifrost.internal.example.com --use
printf '%s\n' "$BIFROST_VIRTUAL_KEY" | bifrost auth set-virtual-key --stdin
bifrost models list                            # models this key may use

bifrost claude --model anthropic/claude-sonnet-4-6
bifrost run codex --model openai/gpt-5
bifrost run gemini --model gemini/gemini-2.5-pro
bifrost run opencode --model openai/gpt-5
```

Docs: https://docs.getbifrost.ai/quickstart/cli/getting-started.md

Use Option B when you need persistent config deployed by MDM.

## Claude Code

Docs: https://docs.getbifrost.ai/cli-agents/claude-code.md

### Static virtual key

`~/.claude/settings.json` (user scope; merge, do not replace):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://bifrost.internal.example.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "sk-bf-your-virtual-key"
  }
}
```

Equivalent shell form: `export ANTHROPIC_BASE_URL=...` and `export ANTHROPIC_AUTH_TOKEN=...`. For one repository only, use `.claude/settings.local.json` (gitignored). Never put a credential in project `.claude/settings.json`; it is committed.

### Key from a vault (recommended for teams)

```bash
#!/bin/bash
# /usr/local/bin/bifrost-credential  (chmod 755; must print ONLY the key)
set -euo pipefail
vault kv get -field=virtual_key "secret/bifrost/devs/$USER"
```

```json
{
  "apiKeyHelper": "/usr/local/bin/bifrost-credential",
  "env": {
    "ANTHROPIC_BASE_URL": "https://bifrost.internal.example.com/anthropic",
    "CLAUDE_CODE_API_KEY_HELPER_TTL_MS": "900000"
  }
}
```

Remove `ANTHROPIC_AUTH_TOKEN` and `ANTHROPIC_API_KEY` from settings and shell when using a helper; both outrank it. Use the full path to the helper. Any macOS keychain or 1Password CLI command that prints the key works the same way.

### Keep a Claude subscription, add gateway governance

Developers sign in with `/login`; the virtual key travels in a header for budgets and logging:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://bifrost.internal.example.com/anthropic",
    "ANTHROPIC_CUSTOM_HEADERS": "x-bf-vk: sk-bf-your-virtual-key"
  }
}
```

Do not set `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_API_KEY` or `apiKeyHelper` in this mode.

### Model mapping (switch or mix providers, data residency)

Claude Code's `opus` / `sonnet` / `haiku` aliases map through env vars. Point them at any provider configured on the gateway:

```json
{
  "env": {
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-8",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "bedrock/global.anthropic.claude-sonnet-4-6",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "vertex/claude-haiku-4-5"
  }
}
```

- Anthropic direct: no prefix (`claude-sonnet-4-6`). Bedrock: `bedrock/...`. Vertex: `vertex/...`. Azure: `azure/...`. Others: `openai/gpt-5.5`, `vertex/gemini-3.1-pro`.
- Remove any specific model ID in the `model` settings key; it overrides the alias mapping.
- Gateway-side alternative: set the aliases to labels such as `sonnet-model` and add a Bifrost routing rule that rewrites them to the target model. You can then change the target for everyone without touching laptops (https://docs.getbifrost.ai/providers/aliasing-models.md).
- Per session: `claude --model bedrock/global.anthropic.claude-sonnet-4-6`, or `/model vertex/claude-haiku-4-5` in a session.
- Non-Claude models may not implement Claude-specific server tools; keep Claude-family models for workflows that need them.
- Do not set `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX`; they bypass `ANTHROPIC_BASE_URL`.

### MCP through the gateway

```bash
claude mcp add-json bifrost '{"type":"http","url":"https://bifrost.internal.example.com/mcp","headers":{"Authorization":"Bearer sk-bf-your-virtual-key"}}'
```

### Verify and pitfalls

- `/status` must show the gateway `Anthropic base URL` and `ANTHROPIC_AUTH_TOKEN` or `apiKeyHelper`, not a claude.ai login method.
- Startup warning about two credentials: run `/logout`.
- `400` mentioning `context_management` or `Extra inputs are not permitted` from a non-Anthropic upstream: set `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1`.
- Managed `forceLoginMethod` / `forceLoginOrgUUID` blocks gateway credentials at startup.
- Claude Code sends `x-claude-code-session-id`; Bifrost uses it for session affinity, so prompt-cache hits are preserved without configuration.
- Managed settings for enforcement: see SKILL.md step 5.

## Codex CLI

Docs: https://docs.getbifrost.ai/cli-agents/codex-cli.md

Run `/logout` in Codex first if the developer ever signed in with an OpenAI account; an existing OAuth session takes precedence over custom credentials.

```bash
export OPENAI_API_KEY="sk-bf-your-virtual-key"
```

`~/.codex/config.toml`:

```toml
model = "openai/gpt-5"
model_provider = "bifrost"

[model_providers.bifrost]
name = "Bifrost"
base_url = "https://bifrost.internal.example.com/openai/v1"
wire_api = "responses"
env_key = "OPENAI_API_KEY"
requires_openai_auth = false
supports_websockets = false
```

- The trailing `/v1` is required (Codex appends `/responses`).
- A named `model_providers` entry is required. Setting only `OPENAI_BASE_URL` can leave Codex on its built-in provider.
- `supports_websockets = false` keeps requests on HTTP so the gateway can translate to non-OpenAI providers.
- To avoid clobbering a developer's personal `OPENAI_API_KEY`, set `env_key` to a dedicated name such as `BIFROST_API_KEY` and export that instead.
- Select models at launch (`codex --model anthropic/claude-opus-4-6`); the in-session `/model` picker uses Codex's own catalog. Confirm with `/status`.
- Non-OpenAI models through Codex are compatibility targets; test each with a real task.
- CI: `codex exec "<task>"` with the same config and the key from a CI secret.

## Gemini CLI

Docs: https://docs.getbifrost.ai/cli-agents/gemini-cli.md

Select **Use Gemini API Key** in Gemini CLI (not Google login or Vertex), then:

```bash
export GEMINI_API_KEY="sk-bf-your-virtual-key"
export GOOGLE_GEMINI_BASE_URL="https://bifrost.internal.example.com/genai"
gemini --model gemini/gemini-2.5-pro
```

- Other providers: `gemini --model anthropic/claude-sonnet-4-6` or `openai/gpt-5`, provided the model supports function calling and streaming.
- Requests going to Google directly mean the auth mode is wrong or the base URL does not end in `/genai`.
- Google account OAuth, ADC and Vertex service accounts do not authenticate to the gateway; only the virtual key does.

## Cursor

Docs: https://docs.getbifrost.ai/cli-agents/cursor.md

1. Cursor Settings (Cmd/Ctrl + ,) > **Models**.
2. **OpenAI API Key**: the developer's virtual key.
3. Toggle **Override OpenAI Base URL** on and enter `https://bifrost.internal.example.com/cursor`.
4. Add models in `provider/model` form, e.g. `anthropic/claude-sonnet-4-5-20250929`, `openai/gpt-5`, `bedrock/<model-id>`.

- Cursor's backend calls the URL, so the gateway must be publicly reachable over HTTPS. Protect it with `enforce_auth_on_inference` and per-developer keys.
- The base URL override is global to all OpenAI-compatible models in Cursor.
- Assign capable tool-calling models to Agent and inline edit; non-tool models only work for chat.
- Requests to Cursor's own hosted models (under a Cursor plan) are billed and routed by Cursor, not by your gateway; decide in policy whether those are allowed, and check `/logs` to confirm which models actually reach the gateway.
- MCP: add the gateway's `/mcp` endpoint in Cursor's MCP settings (`~/.cursor/mcp.json` or project `.cursor/mcp.json`) with the virtual key as a header; keep credentials out of committed project files.

## GitHub Copilot

Docs: https://docs.getbifrost.ai/cli-agents/github-copilot.md (BYOK; labels shift between releases, update first)

### Copilot CLI

```bash
export COPILOT_PROVIDER_TYPE=openai
export COPILOT_PROVIDER_BASE_URL=https://bifrost.internal.example.com/openai/v1
export COPILOT_PROVIDER_API_KEY=sk-bf-your-virtual-key
export COPILOT_MODEL=openai/gpt-5
copilot                                   # or: copilot --model anthropic/claude-sonnet-4-5-20250929
```

Anthropic surface instead: `COPILOT_PROVIDER_TYPE=anthropic`, `COPILOT_PROVIDER_BASE_URL=https://bifrost.internal.example.com/anthropic`. Models need tool calling, streaming and roughly 128k context.

MCP:

```bash
copilot mcp add --transport http --header "x-bf-vk: sk-bf-your-virtual-key" bifrost https://bifrost.internal.example.com/mcp
```

### Copilot app

Settings > Model Providers > Add Provider > **OpenAI-compatible**: Base URL `https://bifrost.internal.example.com/openai/v1/chat/completions`, API key = virtual key. (Or **Anthropic** provider with `https://bifrost.internal.example.com/anthropic`.) MCP: Settings > MCP Servers > Add Server, type HTTP, URL `<gw>/mcp`, header `x-bf-vk: <key>`.

### Copilot Chat in VS Code

Command Palette > **Chat: Manage Language Models** > Add Models > **Custom Endpoint**, then in `chatLanguageModels.json`:

```json
[
  {
    "name": "Bifrost",
    "vendor": "customendpoint",
    "apiKey": "${input:bifrostApiKey}",
    "apiType": "chat-completions",
    "models": [
      {
        "id": "anthropic/claude-sonnet-4-5-20250929",
        "name": "Claude Sonnet 4.5 (Bifrost)",
        "url": "https://bifrost.internal.example.com/openai/v1/chat/completions",
        "toolCalling": true,
        "vision": true,
        "maxInputTokens": 200000,
        "maxOutputTokens": 8192
      }
    ]
  }
]
```

VS Code prompts for the key and keeps it in secret storage. MCP in `.vscode/mcp.json` or user MCP config:

```json
{ "servers": { "bifrost": { "type": "http", "url": "https://bifrost.internal.example.com/mcp", "headers": { "x-bf-vk": "<BIFROST_VIRTUAL_KEY>" } } } }
```

Do not commit a real key in `.vscode/mcp.json`; use the user config for headers with secrets.

## OpenCode

Docs: https://docs.getbifrost.ai/cli-agents/opencode.md

`~/.config/opencode/opencode.json`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "bifrost/openai/gpt-5",
  "provider": {
    "bifrost": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Bifrost",
      "options": {
        "baseURL": "https://bifrost.internal.example.com/openai",
        "apiKey": "sk-bf-your-virtual-key"
      },
      "models": {
        "openai/gpt-5": {},
        "anthropic/claude-sonnet-4-6": {},
        "gemini/gemini-2.5-pro": {}
      }
    }
  }
}
```

Model references are `bifrost/<provider>/<model>`. If OpenCode complains about a missing Google or OpenAI key, it is using a native provider; check the footer and select a `bifrost/...` model.

## Other agents

Same pattern (OpenAI-compatible or Anthropic base URL + virtual key):

- Qwen Code: https://docs.getbifrost.ai/cli-agents/qwen-code.md
- Zed: https://docs.getbifrost.ai/cli-agents/zed-editor.md
- Roo Code: https://docs.getbifrost.ai/cli-agents/roo-code.md
- Claude Desktop (Code tab): https://docs.getbifrost.ai/cli-agents/claude-desktop.md
- LibreChat / Open WebUI: https://docs.getbifrost.ai/cli-agents/librechat.md , https://docs.getbifrost.ai/cli-agents/open-webui.md

If a browser-based client fails with CORS errors, add its headers to Bifrost's Allowed Headers (default allows `Content-Type`, `Authorization`, `X-Requested-With`, `X-Stainless-Timeout`, `X-Api-Key`). CLI agents do not need this.

## Final check per agent

1. Run one short task in the agent.
2. Open `<gw>/logs` and confirm the request is there with the developer's virtual key, the expected provider and model.
3. Switch the agent to a model the key does not allow and confirm a `403`-style rejection rather than a silent fallback.

## CI: GitHub Actions example

One virtual key per pipeline, stored as a CI secret, with tools restricted explicitly:

```yaml
# GitHub Actions step (sketch)
- name: Agent review
  env:
    ANTHROPIC_BASE_URL: https://bifrost.internal.example.com/anthropic
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.BIFROST_VK_CI_REVIEW }}
  run: claude -p "Review the diff in this PR for correctness issues" --allowedTools "Read,Grep,Glob"
```
