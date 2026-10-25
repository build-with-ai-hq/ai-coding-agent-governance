# Rollout checklist

A phased plan for rolling out coding agents with central control. Each phase has an exit criterion; do not widen the rollout until it is met. Owners are suggestions: Platform (gateway and keys), Security (policy, data, MCP), IT (device management), Eng leads (adoption), Finance (budgets).

## Phase 0: Baseline (1 week)

- [ ] Inventory agents in use: run the Quick diagnosis commands from SKILL.md on a sample of machines; ask teams which agents and MCP servers they use.
- [ ] Collect the last three provider invoices and per-key usage exports. Note which keys are shared.
- [ ] List repositories by sensitivity (public, internal, regulated, customer-data-adjacent).
- [ ] List existing MCP servers in use and who owns each.
- [ ] Record baseline delivery metrics per team: PR cycle time, review rework, change-failure rate.

Exit: you know who uses what, what it costs today, and which repositories need stricter rules.

## Phase 1: Policy (in parallel with Phase 0)

Fill in this template and publish it where developers look.

```markdown
# AI coding agent policy

Approved agents: Claude Code, Codex CLI, Cursor, ... (versions >= ...)
Approved providers and regions: e.g. Anthropic via Bedrock (eu-central-1), OpenAI direct
Approved models: <list>; default model per agent: <list>
How to connect: all agents use the company gateway with your personal key. Provider keys are not issued to individuals.
Repositories: allowed everywhere except <regulated repos>; for those, only <provider/region> and no content logging.
Never send: production data, customer PII, secrets, credentials, private keys.
Budgets: default <$X/day, $Y/month> per developer; team budget <$Z/month>; raise via <process>.
MCP servers: only those exposed through the gateway's /mcp endpoint; request new ones via <process>.
Agent permissions: permission prompts stay on; bypass mode is disabled on company machines.
Logging: requests are logged with your identity, model, tokens and cost; prompt content is <stored for N days | not stored>.
CI usage: one key per pipeline, owned by the repo's team, with a budget and expiry.
Exceptions and questions: <channel/owner>
```

- [ ] Data classification decision for prompt/completion content: store (and for how long) or metadata only.
- [ ] Contracts checked: provider data retention and training terms for every approved provider.
- [ ] Policy approved by Security and Eng leadership.

## Phase 2: Gateway (Platform)

- [ ] Deploy Bifrost on an internal HTTPS host with a pinned version, persistent `/app/data`, Postgres log store for shared use.
- [ ] Enable dashboard auth; set `enforce_auth_on_inference: true` so every call needs a virtual key.
- [ ] Add provider keys only for approved providers and regions. Store provider secrets as `env.VAR` references, not literals.
- [ ] Decide content logging globally (`client.disable_content_logging`) and per sensitive team key.
- [ ] Register approved MCP servers on the gateway; mark none as Allow by Default unless every key should see them.
- [ ] If Cursor is approved: expose the gateway publicly over HTTPS (Cursor's backend calls it) and restrict with keys.
- [ ] Prometheus scraping of `/metrics` and a log-retention job matching the policy.
- [ ] Back up the gateway config and database.

Exit: `curl` checks from `references/per-tool-setup.md` return 200 for an allowed model and 403 for a disallowed one; a test request shows in `/logs`.

## Phase 3: Keys and configuration (Platform + IT)

- [ ] Create teams with monthly budgets.
- [ ] Provision one virtual key per developer from the roster (`references/virtual-keys-per-developer.md`), with daily + monthly budgets, a request rate limit, model allowlist, MCP tool allowlist and owner/team metadata.
- [ ] Store keys in the secret manager; build the `apiKeyHelper` script (Claude Code) or equivalent delivery for other agents.
- [ ] Write per-agent config from `references/per-tool-setup.md`; package it for MDM.
- [ ] Claude Code `managed-settings.json`: `allowedProviders: ["customEndpoint"]`, managed `ANTHROPIC_BASE_URL`, `apiKeyHelper`, `availableModels`, `permissions.disableBypassPermissionsMode`, secret-file `deny` rules, `allowedMcpServers` + `allowManagedMcpServersOnly`.
- [ ] Remove `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` and raw provider keys from managed shell profiles.

## Phase 4: Pilot (one team, 2 weeks)

- [ ] 5 to 15 developers across at least two agents.
- [ ] Each pilot user runs the Verify steps from SKILL.md (`/status`, request visible in `/logs`, disallowed model rejected).
- [ ] Daily review of `/logs` for errors: 401 (key delivery), 403 (allowlist too narrow), budget rejections, provider errors on non-native models.
- [ ] Collect friction: missing models, missing MCP tools, latency complaints.
- [ ] Tune budgets from observed spend (around the 90th percentile of daily spend per developer).

Exit: no unresolved 401/403 issues, every pilot request attributed to a person, spend within forecast, pilot users would not want to go back.

## Phase 5: Team-by-team rollout

- [ ] Onboard one or two teams per week; each team lead owns the team budget.
- [ ] Publish a short onboarding page: install agent, run one command or receive MDM config, verify with `/status`, where to ask.
- [ ] CI: one key per pipeline with small daily budget and `expires_at` for one-off jobs; keys stored as CI secrets.
- [ ] Weekly spend and adoption report to team leads (see Measuring adoption in SKILL.md).

## Phase 6: Close the bypasses

- [ ] Revoke all previously shared provider keys at the providers.
- [ ] Block direct egress to provider APIs from developer networks where feasible, leaving only the gateway.
- [ ] Recurring drift check (monthly): Quick diagnosis commands across a sample of machines; any raw key or unmanaged MCP server is an incident.
- [ ] Offboarding hook: deactivate the leaver's virtual key on their last day.

## Ongoing (monthly)

- [ ] Review model allowlists when providers release new models; test with a real agent task before adding.
- [ ] Review top spenders and budget rejections; adjust limits or defaults (cheaper default model for background tasks).
- [ ] Review MCP requests and the tool allowlists per team.
- [ ] Compare agent-assisted PR metrics with the Phase 0 baseline; report value alongside cost.
- [ ] Re-check agent release notes for changed config keys (Claude Code managed settings, Codex provider config, Copilot BYOK).

## Exit criteria summary

| Area | Done when |
|---|---|
| Identity | 100% of agent requests in gateway logs carry a per-person or per-pipeline key |
| Spend | Every key has a daily and monthly budget; no invoice surprises for two cycles |
| Models | Only allowlisted models appear in logs |
| Data | Only approved providers/regions appear in logs; content logging matches policy |
| Tools | Only gateway-exposed MCP servers load on managed machines |
| Evidence | Logs retained per policy and queryable by person, team and cost center |
