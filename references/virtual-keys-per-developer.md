# Virtual keys per developer

How to give every developer and CI pipeline its own gateway credential with budgets, rate limits, model and MCP allowlists, and how to run the key lifecycle. All calls are Bifrost governance APIs (OSS). Docs: https://docs.getbifrost.ai/features/governance/virtual-keys.md , https://docs.getbifrost.ai/features/governance/budget-and-limits.md

Conventions used below:

```bash
GW=https://bifrost.internal.example.com
AUTH=(-u "$BF_ADMIN:$BF_PASS")          # dashboard admin credentials (Basic auth) once dashboard auth is enabled
```

## Hierarchy

```
Customer (optional: whole company)       budget, rate limit
  └─ Team (platform, payments, ...)      budget, rate limit
       └─ Virtual key per developer       budgets, rate limit, provider/model allowlist, MCP tools, metadata
       └─ Virtual key per CI pipeline     small budget, expiry
```

A request must pass every applicable level: the developer's daily budget, their team's monthly budget and the company cap. A key belongs to one team or one customer, not both.

Design rules:

- One key per human, one per pipeline. Never share. Attribution and revocation depend on it.
- Two budgets per developer key: a daily cap that stops runaway loops, and a monthly cap. Reset durations: `1m 1h 1d 1w 1M 1Q 1Y` (`1m` is a minute, `1M` a month). Add `"calendar_aligned": true` on the key to reset at UTC calendar boundaries instead of rolling windows.
- A request rate limit (requests per minute) catches tight agent loops before they cost much.
- Allowlist models explicitly in `provider_configs[].allowed_models`. An empty `provider_configs` denies every provider.
- Put owner, team and cost center in `metadata`; it is copied onto every log entry and cannot be overridden by request headers.

## Create teams

```bash
curl -s "${AUTH[@]}" -X POST $GW/api/governance/teams \
  -H "Content-Type: application/json" \
  -d '{"name": "platform", "budgets": [{"max_limit": 3000.00, "reset_duration": "1M"}]}' \
  | jq -r '.team.id'
```

## Provision keys from a roster

`roster.csv`:

```
email,team_id,daily_usd,monthly_usd
alice@example.com,<platform-team-id>,20,300
bob@example.com,<payments-team-id>,10,150
```

```bash
#!/usr/bin/env bash
# provision-vks.sh roster.csv  -> writes keys to your secret manager, never to stdout logs
set -euo pipefail
tail -n +2 "$1" | while IFS=, read -r email team daily monthly; do
  body=$(jq -n --arg email "$email" --arg team "$team" --argjson d "$daily" --argjson m "$monthly" '{
    name: ("dev-" + ($email | split("@")[0])),
    team_id: $team,
    metadata: {owner: $email, kind: "developer"},
    provider_configs: [
      {provider: "anthropic", allowed_models: ["claude-sonnet-4-6", "claude-haiku-4-5"], key_ids: ["*"], weight: 1.0}
    ],
    budgets: [{max_limit: $d, reset_duration: "1d"}, {max_limit: $m, reset_duration: "1M"}],
    rate_limit: {request_max_limit: 120, request_reset_duration: "1m"},
    is_active: true
  }')
  resp=$(curl -sf "${AUTH[@]}" -X POST "$GW/api/governance/virtual-keys" -H "Content-Type: application/json" -d "$body")
  vk_id=$(jq -r '.virtual_key.id' <<<"$resp")
  vk_val=$(jq -r '.virtual_key.value' <<<"$resp")
  vault kv put "secret/bifrost/devs/${email%@*}" virtual_key="$vk_val" vk_id="$vk_id" >/dev/null
  echo "provisioned $email -> $vk_id"
done
```

Swap `vault kv put` for your secret manager. Model IDs are examples; use the ones your policy approves. The same structure can be declared in `config.json` under `governance.virtual_keys` (with `"value": "env.SOME_VAR"`) if you manage the gateway as code.

Python equivalent for a provisioning service:

```python
import os, requests

GW = os.environ["BIFROST_URL"]
AUTH = (os.environ["BF_ADMIN"], os.environ["BF_PASS"])

def create_dev_key(email: str, team_id: str, daily: float, monthly: float) -> dict:
    body = {
        "name": f"dev-{email.split('@')[0]}",
        "team_id": team_id,
        "metadata": {"owner": email, "kind": "developer"},
        "provider_configs": [{"provider": "anthropic", "allowed_models": ["claude-sonnet-4-6"], "key_ids": ["*"], "weight": 1.0}],
        "budgets": [{"max_limit": daily, "reset_duration": "1d"}, {"max_limit": monthly, "reset_duration": "1M"}],
        "rate_limit": {"request_max_limit": 120, "request_reset_duration": "1m"},
        "is_active": True,
    }
    r = requests.post(f"{GW}/api/governance/virtual-keys", json=body, auth=AUTH, timeout=30)
    r.raise_for_status()
    vk = r.json()["virtual_key"]
    return {"id": vk["id"], "value": vk["value"]}
```

TypeScript:

```typescript
const GW = process.env.BIFROST_URL!;
const auth = "Basic " + Buffer.from(`${process.env.BF_ADMIN}:${process.env.BF_PASS}`).toString("base64");

export async function createDevKey(email: string, teamId: string, daily: number, monthly: number) {
  const res = await fetch(`${GW}/api/governance/virtual-keys`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: auth },
    body: JSON.stringify({
      name: `dev-${email.split("@")[0]}`,
      team_id: teamId,
      metadata: { owner: email, kind: "developer" },
      provider_configs: [{ provider: "anthropic", allowed_models: ["claude-sonnet-4-6"], key_ids: ["*"], weight: 1.0 }],
      budgets: [{ max_limit: daily, reset_duration: "1d" }, { max_limit: monthly, reset_duration: "1M" }],
      rate_limit: { request_max_limit: 120, request_reset_duration: "1m" },
      is_active: true,
    }),
  });
  if (!res.ok) throw new Error(`create VK failed: ${res.status} ${await res.text()}`);
  const { virtual_key } = await res.json();
  return { id: virtual_key.id as string, value: virtual_key.value as string };
}
```

## Mixing providers and residency on one key

A key can allow several providers. Weights give weighted routing, and the remaining providers on the key act as fallbacks. Per-provider budgets cap spend on an expensive provider without blocking the others.

```json
"provider_configs": [
  {"provider": "bedrock", "allowed_models": ["global.anthropic.claude-sonnet-4-6"], "key_ids": ["*"], "weight": 0.8},
  {"provider": "openai", "allowed_models": ["gpt-5"], "key_ids": ["*"], "weight": 0.2,
   "budgets": [{"max_limit": 50.00, "reset_duration": "1M"}]}
]
```

For strict residency, list only the regional provider (for example Bedrock in the approved region) and no direct provider. `key_ids` can pin a key to specific provider credentials (for example the EU Bedrock account) instead of `["*"]`.

## MCP tool allowlist on the key

No `mcp_configs` = no MCP tools (except MCP clients marked Allow by Default). Grant per client:

```bash
curl -s "${AUTH[@]}" -X PUT $GW/api/governance/virtual-keys/<vk_id> \
  -H "Content-Type: application/json" \
  -d '{"mcp_configs": [
        {"mcp_client_name": "github", "tools_to_execute": ["get_issue", "list_pull_requests"]},
        {"mcp_client_name": "docs-search", "tools_to_execute": ["*"]}
      ]}'
```

`["*"]` = all current and future tools of that client; `[]` = none. Enforced both when tools are listed and when they execute. Docs: https://docs.getbifrost.ai/features/governance/mcp-tools.md

## Content logging per key

```bash
curl -s "${AUTH[@]}" -X PUT $GW/api/governance/virtual-keys/<vk_id> \
  -H "Content-Type: application/json" -d '{"disable_content_logging": true}'
```

`true` keeps usage metadata (tokens, cost, latency) but not prompts/completions; `null` inherits the gateway setting.

## CI and contractor keys: expiry

```bash
curl -s "${AUTH[@]}" -X POST $GW/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ci-repo-api-review",
    "metadata": {"kind": "ci", "repo": "api", "owner": "platform@example.com"},
    "provider_configs": [{"provider": "anthropic", "allowed_models": ["claude-haiku-4-5"], "key_ids": ["*"], "weight": 1.0}],
    "budgets": [{"max_limit": 5.00, "reset_duration": "1d"}],
    "rate_limit": {"request_max_limit": 30, "request_reset_duration": "1m"},
    "expires_at": "2026-12-31T23:59:59Z",
    "is_active": true
  }'
```

An expired key is rejected for both inference and MCP tool calls and stays visible for audit. `delete_after_expire: true` lets a daily job remove it.

## Lifecycle

| Event | Action |
|---|---|
| Joiner | Run provisioning for the new roster row; deliver the key via the secret manager / `apiKeyHelper` path. |
| Team change | `PUT` the key with the new `team_id` and `metadata.team`. Old log entries keep the old attribution. |
| Budget raise | `PUT` new `budgets`. Prefer temporary raises with a reminder to revert. |
| Suspected leak | `PUT {"is_active": false}` immediately, issue a new key, check `/logs` for the old key's recent traffic. |
| Rotation | Create the new key, update the vault entry (helpers pick it up within the helper TTL), then deactivate the old key. |
| Leaver | Deactivate on the offboarding day (`is_active: false`), delete after your log-retention window. Wire this into your HR/IdP offboarding flow. |

```bash
curl -s "${AUTH[@]}" -X PUT $GW/api/governance/virtual-keys/<vk_id> -H "Content-Type: application/json" -d '{"is_active": false}'
curl -s "${AUTH[@]}" $GW/api/governance/virtual-keys | jq '.'                  # inventory
curl -s "${AUTH[@]}" "$GW/api/governance/virtual-keys?metadata_owner=alice@example.com" | jq '.'
```

## Spend reports

- UI: Virtual Keys page shows current budget usage; each metadata entry has a **View spend** link into filtered logs.
- API: `GET /api/logs?metadata_<key>=<value>` filters logs by key metadata, e.g. `metadata_cost_center=cc-1234`.
- Prometheus: `/metrics` exposes `bifrost_cost_total`, `bifrost_input_tokens_total`, `bifrost_output_tokens_total`. Inspect available labels on your instance (`curl -s -u ... $GW/metrics | grep bifrost_cost_total | head`) before building dashboards; add `x-bf-dim-<name>` request headers from agents that support custom headers for extra labels.

Weekly report to send team leads: active keys, spend per key versus budget, keys that hit a budget or rate limit, top models by cost. Use it to tune limits at around the 90th percentile of real daily spend.
