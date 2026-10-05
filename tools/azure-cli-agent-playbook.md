# Azure CLI investigation playbook for coding agents

A procedure for any agent (or person) that uses `az` to find out what exists in an Azure subscription, written after an agent reported "your account has no access" because its own `--query` expression was malformed. It ships as a skill (`skills/azure-cli/SKILL.md`) in [jblenman/ai-agent-templates](https://github.com/jblenman/ai-agent-templates) for Codex CLI, OpenCode and Claude Code; this is the same text. Facts checked against the Azure CLI docs on 2026-10-05: `datafactory` and `resource-graph` are extensions, `synapse` is core; JMESPath strings are single-quoted or backtick literals and double quotes inside a filter predicate return empty output; query strings are case-sensitive.


The failure this prevents: an agent runs `az … --query "<guess>"`, gets `[]`, and reports "your account has no access". The empty result came from the query, not the permission. Every rule below exists to make that mistake impossible.

## 1. Login and scope — always first

```
az account show -o table          # who am I, which subscription, which tenant
az account list -o table          # every subscription this login can see
az account set --subscription "<name or id>"
```

- `az account show` fails → `az login` (browser), `az login --use-device-code` (no local browser: remote desktop, SSH, locked-down proxy), `az login --tenant <tenant-id>` when the account spans tenants, `az login --identity` on a VM with a managed identity.
- Confirm the subscription in `az account show` matches the one the user means before listing anything. Most "nothing there" results are the wrong subscription.
- Proxies: `HTTPS_PROXY`/`NO_PROXY` must be set in the shell that runs `az`, not only in the browser.

## 2. Inventory without filters, then narrow

```
az resource list -o table                                                   # everything in the subscription
az resource list --resource-type Microsoft.DataFactory/factories -o table   # server-side type filter
az resource list --resource-type Microsoft.Synapse/workspaces -o table
az group list -o table
```

Rules:
- Run the unfiltered command first and note the raw count. Only then add `--resource-group`, `--resource-type`, `--query`.
- Prefer server-side filters (`--resource-group`, `--resource-type`, `--name`) over `--query`; they are case-insensitive and they error on bad input, while `--query` returns `[]` silently.
- Service command groups: `az datafactory` (extension: `az extension add --name datafactory`), `az synapse` (built in), `az storage`, `az sql`, `az keyvault`, `az graph query -q "<KQL>"` (extension `resource-graph`; the fastest cross-subscription inventory). A missing extension prints `'<name>' is misspelled or not recognized by the system` — install it, don't conclude the service is absent.

## 3. `--query` (JMESPath) — the usual cause of a false "nothing found"

- JMESPath never errors on an unknown field name; it returns `null` or `[]`. So look at real field names first: `az resource list -o json --query "[0]"` prints one full item (`name`, `type`, `resourceGroup`, `location`, `id`, `tags`, …; CLI field names are camelCase).
- String literals: use single quotes inside the expression — `[?type=='Microsoft.DataFactory/factories'].name`. Double quotes inside a JMESPath expression mean *identifier*, not string, and backticks mean a JSON literal (and in PowerShell the backtick is the escape character).
- Shell quoting: wrap the whole expression in double quotes in both PowerShell and bash when the literals use single quotes: `--query "[?type=='Microsoft.Synapse/workspaces'].{name:name, rg:resourceGroup}"`.
- Comparisons are case-sensitive and `type` is returned in the provider's casing (`Microsoft.DataFactory/factories`). When unsure, filter with `--resource-type` server-side or use `contains(type, 'DataFactory')` as a first probe.
- Build the expression in steps: `"[0]"` → `"[0].name"` → `"[].name"` → the filter. Test a filter on an item you have already seen before trusting its empty result.
- A JMESPath *syntax* error does error (`argument --query: invalid jmespath_type value`). An empty `[]` is a *semantic* miss: wrong field, wrong value, wrong case, wrong scope.

## 4. Reading results

- `-o table` hides fields and flattens nested properties; `-o json` is the truth; `-o tsv` for scripting.
- `AuthorizationFailed` / 403 = a real permission limit, quote it verbatim. `ResourceNotFound`, `ResourceGroupNotFound`, `SubscriptionNotFound` = wrong scope. `InvalidResourceType` = wrong provider/type string. Nothing on stderr and `[]` on stdout = your filter.
- Some list commands page; if a count looks suspiciously round or a `nextLink` appears, fetch the next page (`--max-items` / `--next-token` on commands that support them) before reporting a total.
- Warnings about an experimental command group or an extension update are not errors; read past them.

## 5. Hand-back format for an investigation

Report, in this order: the identity and subscription used (`az account show` line), a table of what exists per resource type with the raw counts, the exact commands that produced them, and any limits with their verbatim error text. "No access" appears only next to an `AuthorizationFailed` quote. Separate what was observed from what was inferred.
