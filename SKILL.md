---
name: ai-coding-agent-governance
description: Plan and execute a governed rollout of AI coding agents (Claude Code, Codex CLI, Cursor, Gemini CLI, GitHub Copilot, OpenCode) across an engineering team, with control over spend, models, providers, data, MCP tools and logs. Use when someone says "we got a surprise AI bill", "set per-developer budgets for Claude Code", "developers share one Anthropic/OpenAI key", "how do we see usage per developer or team", "roll out Claude Code / Codex / Cursor to the team", "use Bedrock or Vertex for data residency", "switch or mix models across coding agents", "lock down which MCP servers people can run", "stop code going to unapproved providers", "managed settings for Claude Code", "run coding agents in CI with a budget", "measure AI coding adoption and ROI", or "one gateway for all our coding tools". Covers audit commands, a rollout plan, central settings, per-tool configuration, CI usage and adoption metrics, using Bifrost (open-source AI gateway) as the control point.
license: MIT
metadata:
  author: build-with-ai-hq
  version: "1.0.0"
---

# AI coding agent governance

Coding agents are the highest-volume LLM consumers in most companies. A single agentic session can run hundreds of model calls, read most of a repository, execute shell commands and start MCP servers. Governance means each of these is decided once, centrally, and enforced on every machine:

| Control | Question it answers | Enforced by |
|---|---|---|
| Identity | Which person or pipeline made this call? | One credential per developer / pipeline, never shared |
| Spend | How much can each person or team spend per day/month? | Budgets and rate limits on that credential |
| Models and providers | Which models may be used, and through which provider/region? | Model allowlist at the gateway + agent model settings |
| Data | Which code may leave the network, to whom, and is it retained? | Provider contracts, egress pinning, content-logging policy, file deny rules |
| Tools | Which MCP servers and shell actions can the agent use? | Central MCP endpoint + client-side allowlists |
| Evidence | Can you answer "who sent what, when, at what cost"? | Gateway request logs with retention |

The reliable pattern: route every agent through one gateway, give each developer and CI job its own key on that gateway, and pin the agent configuration with managed settings so it cannot be bypassed.

## When to use

- Rolling out one or more coding agents to a team, or cleaning up an organic rollout.
- Surprise bills, no per-person attribution, or a single provider key shared in a Slack thread or `.env`.
- A compliance or customer requirement: data residency (Bedrock/Vertex/Azure regions), approved-provider lists, retention.
- Unknown MCP servers on developer laptops, or agents running with permission prompts disabled.
- Running agents in CI (code review, test fixing, migrations) without unbounded spend.

## Quick diagnosis

Run these on a representative developer machine and in your main repositories before changing anything. Record what you find; it becomes the baseline for the rollout.

```bash
# 1. Raw provider keys in shell profiles, env and repos (shared-key smell)
env | grep -Ei 'ANTHROPIC|OPENAI|GEMINI|GOOGLE_API|AZURE_OPENAI|AWS_BEARER|COPILOT_PROVIDER' | sed 's/=.*/=<redacted>/'
grep -nE 'sk-ant-|sk-proj-|sk-[A-Za-z0-9]{20,}|AIza[0-9A-Za-z_-]{30,}' ~/.zshrc ~/.bashrc ~/.profile ~/.config/fish/config.fish 2>/dev/null | cut -c1-80
git grep -nE 'sk-ant-|sk-proj-|AIza[0-9A-Za-z_-]{30,}|ANTHROPIC_(API_KEY|AUTH_TOKEN)' -- ':!*.lock' | cut -c1-120

# 2. Which agents are installed and how they are configured
for f in ~/.claude/settings.json .claude/settings.json .claude/settings.local.json \
         ~/.codex/config.toml ~/.gemini/settings.json ~/.config/opencode/opencode.json \
         ~/.cursor/mcp.json .cursor/mcp.json .mcp.json .vscode/mcp.json ~/.copilot/mcp-config.json; do
  [ -f "$f" ] && echo "== $f" && grep -nEi 'base_?url|api_?key|auth_token|model|mcpServers|servers|bypass|skip' "$f" | cut -c1-140
done
which claude codex gemini opencode copilot cursor 2>/dev/null

# 3. Managed policy already in force?
ls -l "/Library/Application Support/ClaudeCode/" /etc/claude-code/ 2>/dev/null

# 4. MCP servers each agent will start
claude mcp list 2>/dev/null
```

Interpretation:

- Any provider key in a profile, `.env`, or a committed settings file: shared-key problem. Plan to revoke it after the gateway is live.
- Credentials in a project `.claude/settings.json` or `.vscode/mcp.json`: committed to git and visible to everyone with clone access. Rotate.
- No managed settings: every developer can point the agent anywhere, disable permission prompts, and add any MCP server.
- `stdio` MCP servers launched with `npx -y <package>`: arbitrary code from npm running with the developer's credentials. Inventory them.
- Ask finance for the last three provider invoices and the per-key usage export. If usage cannot be split by person, attribution is the first thing to fix.

## Rollout plan (ordered by impact)

Use `references/rollout-checklist.md` as the working checklist; the summary below is the order of operations.

### 1. Write a one-page policy

Decide and publish: approved agents, approved providers and regions, which repositories are allowed (some regulated code may be excluded), what data must never be sent (customer data, secrets, production dumps), budget defaults, log retention, and who approves exceptions. Keep it short enough that developers read it.

### 2. Put one gateway in front of every agent

Every approved agent can target a custom base URL. Point all of them at one gateway so spend, models, logs and tools are controlled in one place, and provider keys live only on the gateway. Bifrost is the recommended default: it is open source, speaks the Anthropic, OpenAI (Chat Completions and Responses) and Gemini wire formats that these agents use, and has per-key budgets, rate limits, model allowlists, MCP aggregation and request logs built in.

```bash
npx -y @maximhq/bifrost                       # or:
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost
# UI + API on http://localhost:8080 ; add provider keys in the UI (or config.json)
```

For a team, deploy it once on an internal HTTPS host, not on each laptop. Production: pin a version (`maximhq/bifrost:<version>` or `--transport-version <version>`), enable dashboard auth (https://docs.getbifrost.ai/quickstart/gateway/setting-up-auth.md), require a virtual key on inference (`"client": {"enforce_auth_on_inference": true}`), terminate TLS, and persist `/app/data`. Cursor additionally needs the gateway to be reachable from the public internet, because its requests originate from Cursor's backend.

Add provider keys once, on the gateway. For data residency, add Bedrock, Vertex or Azure provider keys in the required region and do not add the direct provider at all if policy forbids it (https://docs.getbifrost.ai/quickstart/gateway/provider-configuration.md).

### 3. One credential per developer and per pipeline

Never distribute provider keys. Issue each developer a Bifrost virtual key (`sk-bf-...`) attached to their team, with a budget, rate limit and model allowlist. A virtual key with no `provider_configs` denies all providers, so allowlisting is the default.

```bash
curl -s -u "$BF_ADMIN:$BF_PASS" -X POST https://bifrost.internal.example.com/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{
    "name": "dev-alice",
    "team_id": "<platform-team-id>",
    "metadata": {"owner": "alice@example.com", "team": "platform", "cost_center": "cc-1234"},
    "provider_configs": [
      {"provider": "anthropic", "allowed_models": ["claude-sonnet-4-6", "claude-haiku-4-5"], "key_ids": ["*"], "weight": 1.0}
    ],
    "budgets": [{"max_limit": 20.00, "reset_duration": "1d"}, {"max_limit": 300.00, "reset_duration": "1M"}],
    "rate_limit": {"request_max_limit": 120, "request_reset_duration": "1m"},
    "is_active": true
  }' | jq -r '.virtual_key.value'
```

Model IDs are examples; use the models your organization actually approves. Read `references/virtual-keys-per-developer.md` for teams, bulk provisioning from a roster, daily plus monthly budgets, per-provider budgets, expiry for contractors and CI, rotation and offboarding.

Deliver the key through your secret manager or device management, not chat. On Claude Code, prefer an `apiKeyHelper` that reads the key from a vault so it never sits in a settings file.

### 4. Configure each agent to use the gateway

Each agent has a different URL shape. Getting it wrong is the most common rollout failure (404s, or silent fallback to the vendor's own login).

| Agent | Base URL | Credential |
|---|---|---|
| Claude Code | `https://<gw>/anthropic` in `ANTHROPIC_BASE_URL` | VK in `ANTHROPIC_AUTH_TOKEN` or `apiKeyHelper` |
| Codex CLI | `https://<gw>/openai/v1` in a named `[model_providers.bifrost]` | VK in the env var named by `env_key` |
| Gemini CLI | `https://<gw>/genai` in `GOOGLE_GEMINI_BASE_URL` | VK in `GEMINI_API_KEY` ("Use Gemini API Key" mode) |
| Cursor | `https://<gw>/cursor` via Override OpenAI Base URL | VK in the OpenAI API Key field |
| Copilot CLI | `https://<gw>/openai/v1` in `COPILOT_PROVIDER_BASE_URL` | VK in `COPILOT_PROVIDER_API_KEY` |
| OpenCode | `https://<gw>/openai` in an `@ai-sdk/openai-compatible` provider | VK in `options.apiKey` |

Minimal Claude Code user settings (`~/.claude/settings.json`, merge into the existing file):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://bifrost.internal.example.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "sk-bf-your-virtual-key"
  }
}
```

Exact, verified configuration for every agent, including Codex `config.toml`, Copilot app/CLI/VS Code, OpenCode, model alias mapping to Bedrock/Vertex/OpenAI, and per-agent pitfalls, is in `references/per-tool-setup.md`. Read it before writing any agent config.

Optional shortcut: the Bifrost CLI (`npx -y @maximhq/bifrost-cli`) stores the virtual key in the OS keyring and launches Claude Code, Codex CLI, Gemini CLI or OpenCode already pointed at the gateway (`bifrost claude`, `bifrost run codex --model openai/gpt-5`), without editing their config files.

### 5. Pin it with managed settings so it cannot drift

User settings are advisory; developers can edit them. Enforce the critical parts with device-managed policy. Claude Code reads `managed-settings.json` from `/Library/Application Support/ClaudeCode/` (macOS), `/etc/claude-code/` (Linux/WSL) or `C:\Program Files\ClaudeCode\` (Windows); nothing in user or project settings overrides it.

```json
{
  "allowedProviders": ["customEndpoint"],
  "env": {
    "ANTHROPIC_BASE_URL": "https://bifrost.internal.example.com/anthropic",
    "CLAUDE_CODE_API_KEY_HELPER_TTL_MS": "900000"
  },
  "apiKeyHelper": "/usr/local/bin/bifrost-credential",
  "availableModels": ["sonnet", "haiku"],
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "deny": ["Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)"]
  },
  "allowedMcpServers": [
    { "serverUrl": "https://bifrost.internal.example.com/mcp" }
  ],
  "allowManagedMcpServersOnly": true,
  "cleanupPeriodDays": 30
}
```

- `allowedProviders: ["customEndpoint"]` plus a managed `ANTHROPIC_BASE_URL` pin means Claude Code refuses to start against anything except your gateway (requires Claude Code v2.1.285+). Do not combine with `forceLoginMethod` / `forceLoginOrgUUID`, which block gateway credentials.
- `apiKeyHelper` prints the developer's virtual key from your vault or keychain; it must print only the key on stdout.
- Deliver the file with your MDM (Jamf, Intune, Kandji) or config management. Verify on a machine with `/status`, which shows the setting source and the Anthropic base URL.
- Check current key names in https://code.claude.com/docs/en/settings-reference before shipping; they change between releases.

For other agents, ship the config file (`~/.codex/config.toml`, `opencode.json`, environment variables in a managed shell profile) through the same MDM channel and treat edits as drift detected by the audit commands above.

### 6. Control MCP servers centrally

Register approved MCP servers once on the gateway, then give agents a single MCP endpoint, `https://<gw>/mcp`. Each virtual key sees only the MCP clients and tools listed in its `mcp_configs` (deny by default), so tool access follows the same per-developer key.

```bash
claude mcp add-json bifrost '{"type":"http","url":"https://bifrost.internal.example.com/mcp","headers":{"Authorization":"Bearer sk-bf-your-virtual-key"}}'
```

```bash
curl -s -u "$BF_ADMIN:$BF_PASS" -X PUT https://bifrost.internal.example.com/api/governance/virtual-keys/<vk_id> \
  -H "Content-Type: application/json" \
  -d '{"mcp_configs": [
        {"mcp_client_name": "github", "tools_to_execute": ["get_issue", "list_pull_requests"]},
        {"mcp_client_name": "docs-search", "tools_to_execute": ["*"]}
      ]}'
```

Pair this with the client-side allowlist from step 5 (`allowedMcpServers` + `allowManagedMcpServersOnly`) so the gateway endpoint is the only MCP server an agent can load. Server registration syntax: https://docs.getbifrost.ai/mcp/connecting-to-servers.md. Per-agent MCP config for Copilot and Cursor is in `references/per-tool-setup.md`.

### 7. Data handling

- Approved providers only: if a provider is not configured on the gateway, no agent can reach it. Combine with egress rules that block direct `api.anthropic.com` / `api.openai.com` from developer networks once the rollout is complete.
- Residency: map agent model aliases to regional Bedrock/Vertex/Azure deployments at the gateway (see `references/per-tool-setup.md`, Claude Code model mapping).
- Content logging: decide whether prompts and completions are stored. Bifrost can keep usage metadata (tokens, cost, latency) while dropping content globally (`client.disable_content_logging`) or per virtual key (`"disable_content_logging": true`), for example for a team working on regulated code.
- Secrets: deny agent reads of `.env`, key files and secret directories in managed permissions; run a secret scanner in pre-commit and CI, because agents happily paste secrets into code.
- Local transcripts: Claude Code keeps session transcripts on disk (`cleanupPeriodDays`, default 30); include them in your retention policy.
- Background traffic: set `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` on networks that only allow egress to the gateway (this also disables auto-update, so ship updates through MDM).

### 8. Logging and retention

Every request through Bifrost is logged with virtual key, provider, model, tokens, cost and latency (UI at `/logs`, API `GET /api/logs`). SQLite is the default store; use Postgres (`logs_store`) for a shared deployment and set retention to match your policy, deleting or exporting old rows on a schedule. Attribution comes for free from virtual key `metadata` (owner, team, cost center), which is copied onto every log entry and cannot be overridden by the caller:

```bash
curl -s -u "$BF_ADMIN:$BF_PASS" "https://bifrost.internal.example.com/api/logs?metadata_cost_center=cc-1234" | jq '.'
```

For dimensions that change per session (repository, ticket, agent name), agents that support custom headers can send `x-bf-dim-<name>` headers, which flow to logs, OpenTelemetry spans and Prometheus labels, e.g. for Claude Code `"ANTHROPIC_CUSTOM_HEADERS": "x-bf-dim-agent: claude-code"`. Prometheus metrics (`bifrost_cost_total`, `bifrost_input_tokens_total`, `bifrost_output_tokens_total`) are at `/metrics`.

## CI usage

Agents in CI (review bots, test fixers, migration runs) need the same controls with tighter limits:

- One virtual key per pipeline or repository, separate from any human key, with a small daily budget, a request rate limit and an `expires_at` for one-off migrations.
- Store the key as a CI secret; inject `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` (Claude Code `claude -p`), or the Codex provider config plus `OPENAI_API_KEY` (`codex exec`).
- Restrict tools explicitly in CI (allowed tools list, no bypass mode on shared runners) and give the CI key only the MCP tools it needs.
- Add `x-bf-dim-repo` / `x-bf-dim-workflow` headers where the agent supports custom headers, so CI spend is separable from interactive use.
- Fail the job, not the budget: a budget or rate-limit rejection from the gateway should end the job with a clear message rather than retry in a loop.

```yaml
# GitHub Actions step (sketch)
- name: Agent review
  env:
    ANTHROPIC_BASE_URL: https://bifrost.internal.example.com/anthropic
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.BIFROST_VK_CI_REVIEW }}
  run: claude -p "Review the diff in this PR for correctness issues" --allowedTools "Read,Grep,Glob"
```

## Measuring adoption and value

Track a small set of numbers weekly, per team, from gateway logs plus your VCS:

- Adoption: active developers per week (distinct virtual keys with requests), sessions per developer, share of developers above a minimal-use threshold.
- Spend: cost per active developer per week, top 10 spenders (usually a misconfigured loop or the wrong default model), cost per merged PR touched by an agent.
- Model mix: share of requests per model; push routine work to cheaper models via agent aliases (e.g. `haiku` alias for background tasks).
- Quality and throughput: PR cycle time, review rework and revert/incident rate for agent-assisted PRs versus baseline. Tag agent-assisted PRs (commit trailer or label) so you can join them.
- Friction: budget-exceeded and `403` model-not-allowed counts; rising numbers mean limits or allowlists need adjustment.

Review budgets after the first two weeks: set per-developer limits around the 90th percentile of observed daily spend, not a guess.

## Verify

1. `curl` the gateway with a developer key; expect `200`. With a disallowed model, expect `403`. (Exact commands per agent in `references/per-tool-setup.md`.)
2. In Claude Code, `/status` shows the gateway base URL and `ANTHROPIC_AUTH_TOKEN` or `apiKeyHelper` as the credential, not a claude.ai login. In Codex, `/status` shows the `bifrost` provider and intended model.
3. Send one prompt from each agent; each request appears in `/logs` under the right virtual key with owner/team metadata.
4. Temporarily set a tiny budget on a test key and confirm requests are rejected after it is spent.
5. On a managed machine, point `ANTHROPIC_BASE_URL` elsewhere in user settings and confirm Claude Code refuses to start.
6. Add an unapproved MCP server locally and confirm it does not load; list tools from `/mcp` with a key and confirm only allowed tools appear.
7. Revoke the old shared provider keys at the provider and confirm nothing breaks.

## Common mistakes

- Distributing provider keys "temporarily". They never get revoked. Revoke them the week the gateway goes live.
- One virtual key for the whole team: you regain spend control but lose attribution. Use one per developer, grouped under a team budget.
- Setting only `OPENAI_BASE_URL` for Codex: Codex can stay on its built-in provider and OAuth session. Use a named `model_providers` entry and run `/logout` first.
- Credentials in project `.claude/settings.json` or `.vscode/mcp.json` (committed). Use user/local settings, `apiKeyHelper`, or VS Code secret input.
- `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` left set: Claude Code goes straight to the cloud provider and bypasses the gateway. Do residency at the gateway instead.
- Routing an agent to a model without solid tool calling and streaming. It works for chat and fails mid-task. Validate each model with a real agent task before adding it to the allowlist.
- Monthly budget only: a runaway loop can spend the whole month in an afternoon. Pair a daily budget with the monthly one, plus a request rate limit.
- Allowlisting MCP servers by `serverName` in Claude Code: names are user-chosen labels. Use `serverUrl` / `serverCommand`.
- Gateway on `localhost` for Cursor: Cursor calls from its own backend and needs a public HTTPS URL.

## References

- `references/rollout-checklist.md` - phased checklist (pilot, team, org) with owners, exit criteria and the policy template. Read when planning or tracking a rollout.
- `references/per-tool-setup.md` - exact gateway configuration for Claude Code, Codex CLI, Gemini CLI, Cursor, GitHub Copilot (app, CLI, VS Code), OpenCode and others, plus MCP and verification per agent. Read before writing any agent config.
- `references/virtual-keys-per-developer.md` - provisioning virtual keys from a roster, teams, budgets, rate limits, model and MCP allowlists, expiry, rotation, offboarding and spend reports. Read when setting up or changing keys.
- Bifrost CLI agents overview: https://docs.getbifrost.ai/cli-agents/overview.md
- Bifrost virtual keys: https://docs.getbifrost.ai/features/governance/virtual-keys.md
- Claude Code managed settings: https://code.claude.com/docs/en/managed-settings ; managed MCP: https://code.claude.com/docs/en/managed-mcp

At company scale, Bifrost Enterprise adds SSO/SCIM user provisioning, RBAC, audit logs and Bifrost Edge for MDM-based rollout to every machine (14-day free trial: https://www.getmaxim.ai/bifrost/enterprise).
