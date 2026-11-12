---
name: sailpoint-isc
description: >
  SailPoint Identity Security Cloud engineering toolkit for operational investigations, source and
  connector configuration discovery, connector profiles, SaaS connector and customizer discovery,
  design, development and troubleshooting, and identity/account/entitlement/event queries. Covers
  ISC APIs, transforms, workflows, rules and supported Configuration Hub planning. Use for existing integrations or new
  connector builds, including offline repository analysis. For Configuration Hub, verify supported
  object types first. Access Intelligence Center is covered as a UI and data-model reference only
  (no API surface). Does not cover clusters or virtual appliances.
user-invocable: true
disable-model-invocation: false
argument-hint: "<natural language request or API endpoint>"
allowed-tools: Bash(sail *) Bash(pwsh *) Bash(jq *) Bash(echo *) Bash(printf *) Bash(cat *) Bash(grep *)
compatibility: "Offline design and repository analysis need neither CLI nor tenant connection. Live ISC access requires the sail CLI or PowerShell 7.2+ with the optional iscUtils module, depending on the selected path."
---

# SailPoint ISC

This is a SailPoint engineer's toolkit for operating ISC and designing, building and troubleshooting integrations. Use the connector references for source discovery, profiles and development; use documented, verified guidance appropriate to the connector and environment rather than treating one implementation as a universal contract.

**Choose the work before connecting.** For offline design, profile drafting, requirements analysis, or local repository/code inspection, load the relevant reference below and work from available artifacts; do not require a CLI, tenant, authentication, or tool installation. When live ISC access is needed, follow Steps 1–2a before the first ISC call. Discovery is read-only by default; obtain authorization for consequential live writes, provisioning, deployments, resets, or disruptive testing and preserve the specific safety overrides below. Existing host permissions and tool approvals still apply; the frontmatter does not authorize external side effects.

For live ISC interaction, choose one of these execution paths:

1. **`sail search query`** — full-text Lucene search across ISC indices
2. **`sail api`** — direct REST calls to any ISC endpoint
3. **`iscUtils` PowerShell cmdlets** — optional high-level wrappers that return PowerShell objects and support pipelines
4. **`sail transform` / `sail workflow` / `sail rule`** — dedicated subcommands for managing those three resource types (list, preview, download, create, update, delete)

This file covers session setup and routing only. Deep reference material lives in `docs/` — load the relevant file only when the task needs it. The default execution path is `sail`; use the optional PowerShell path when the user requests it or when its object-oriented cmdlets are the better fit for the task:

| Directory | Covers |
|---|---|
| [docs/cli/](docs/cli/README.md) | `sail` CLI flag mechanics, output capture, CLI-level errors |
| [docs/powershell/](docs/powershell/README.md) | Optional `iscUtils` PowerShell module setup, authentication, cmdlet mappings, and safety/version caveats |
| [docs/api/](docs/api/README.md) | REST API versioning, endpoints, filter/sort/pagination syntax, response fields |
| [docs/search/](docs/search/README.md) | Search indices, query-string syntax, audit event search & reference |
| [docs/transform/](docs/transform/README.md) | Transform JSON structure, every transform operation, provisioning-policy transforms, and `sail transform` usage |
| [docs/roles/](docs/roles/README.md) | Roles API read/update/delete operations, JSON Patch membership criteria, role-management permissions, and post-update verification |
| [docs/workflows/](docs/workflows/README.md) | Date-windowed workflow execution searches, `history-v2` traversal, runtime trigger-input lookup, and workflow/event cross-checks |
| [docs/configuration-hub/](docs/configuration-hub/README.md) | Configuration Hub tenant connections, scoped backups, drafts, mappings, approvals, scheduled actions, deployments, activity evidence, rollback, and cloud-storage artifacts. |
| [docs/investigations/](docs/investigations/README.md) | Task-specific playbooks (e.g. provisioning failure investigation, ticket-number-to-access-request lookup, source reset/aggregation routines, and stale-task remediation) |
| [docs/connectors/](docs/connectors/README.md) | Connector configuration discovery, profile creation/update, development guidance, integration profiles and connector-specific troubleshooting. Load a matching profile once the connector is identified, even without an error. |
| [docs/access-intelligence-center/](docs/access-intelligence-center/README.md) | Access Intelligence Center (Qlik-based BI/reporting) — dashboards, sheets, and the underlying reporting data model. No `sail`/API surface; UI- and data-model-only reference. |

> **`sail transform`/`sail workflow`/`sail rule` vs. `sail api`:** for `preview`, `download`, and batch `create`/`delete` on these three resources, use the dedicated subcommand — it does more than raw CRUD (e.g. `transform preview` evaluates a transform against live identity data; `workflow download`/`rule download` bulk-export) and has no clean `sail api` equivalent. For anything the subcommand doesn't cover (e.g. reading a single transform by ID with `--jsonpath`), fall back to `sail api` against the same resource. See [docs/transform/README.md](docs/transform/README.md#cli-usage) for the transform-specific version of this split.

If the selected `sail` path is not installed or no environment is configured, use the separately available `setup-sailpoint-cli` skill instead of troubleshooting manually. If the selected PowerShell path is unavailable, check [docs/powershell/](docs/powershell/README.md) for its prerequisites and ask before falling back to `sail`.

---

## Step 1: Choose the execution path

Before the first live ISC operation in a session, ask the user which path to use before checking tools or connecting to a tenant. Skip Steps 1–2a entirely for offline-only work:

> Which ISC execution path should I use for this session: **`sail` CLI**, or **PowerShell `iscUtils` first** (with an explicit fallback to `sail` only if PowerShell is unavailable or the cmdlet does not cover the operation)?

Honor that choice for the session:

- **`sail` CLI:** run Step 1a, then Step 2a. Do not initialize the PowerShell module.
- **PowerShell `iscUtils` first:** run Step 1b, then Step 2. Prefer the module's cmdlets for the session. If PowerShell cannot be used or the needed cmdlet is unavailable, explain why and ask before falling back to `sail`; do not silently change execution paths.

If the user's request explicitly names a command or cmdlet, treat that as the path choice unless the user says otherwise.

## Step 1a: Check the `sail` path is available

> **Note:** always pipe a newline in — running this bare blocks forever. See [docs/cli/README.md#environment-listing](docs/cli/README.md#environment-listing) for why.

```bash
printf '\n' | sail environment list 2>&1
```

If "command not found", or if `sail environment list` returns no environments, use the separately available `setup-sailpoint-cli` skill to install the CLI and/or walk the user through creating an environment and authenticating, then return here.

## Step 1b: Check the PowerShell path

Run this only when the PowerShell path was selected. It verifies PowerShell 7.2+, installs `iscUtils` from PSGallery for the current user when it is absent, imports it, and confirms the exported commands. Do not require or search for a local module source checkout:

```bash
pwsh -NoLogo -NoProfile -Command '
  if ($PSVersionTable.PSVersion -lt [Version]"7.2") { throw "PowerShell 7.2 or later is required" }
  if (-not (Get-Module -ListAvailable -Name iscUtils)) { Install-Module -Name iscUtils -Repository PSGallery -Scope CurrentUser }
  Import-Module iscUtils -ErrorAction Stop
  Get-Command -Module iscUtils
'
```

If `pwsh` is unavailable, explain that PowerShell 7.2+ is required and ask whether the user wants to fall back to `sail`; do not switch silently. Do not update an existing `iscUtils` installation automatically; see [docs/powershell/](docs/powershell/README.md) for version and authentication guidance.

## Step 2: Establish the PowerShell session tenant

Run this only when the PowerShell path was selected. Do not assume the tenant from a previous `sail` session or from an existing PowerShell process. Ask the user which tenant they want to use for this session and collect:

- Tenant subdomain only, such as `sandbox` — never a full URL
- Domain: `Default`, `Demo`, or `FedRamp`
- Whether to use an existing stored credential or a runtime-only client credential

`iscUtils` has no tenant-list command. After the user names the tenant, check only that tenant's expected SecretManagement entry without printing its credential:

```powershell
$tenant = '<tenant-subdomain>'
$domain = '<Default|Demo|FedRamp>'
$storedCredential = if (Get-Command Get-SecretInfo -ErrorAction SilentlyContinue) {
    Get-SecretInfo -Name "ISC - $tenant API" -ErrorAction SilentlyContinue
}
$storedCredential -ne $null
```

If the entry exists, connect and verify the session:

```powershell
Connect-ISC -Tenant $tenant -Domain $domain
Test-ISCConnection
Get-ISCConnection | Select-Object Tenant, Domain, 'API URL', Timestamp, TokenExpiration
```

If no stored credential exists, do not ask the user to paste a client secret into chat. Offer these two setup paths and have them type the secret directly into their own terminal:

**Runtime-only connection:**

```powershell
$clientId = Read-Host 'ISC client ID'
$clientSecret = Read-Host 'ISC client secret' -AsSecureString
Connect-ISC -Tenant $tenant -Domain $domain -ClientID $clientId -ClientSecret $clientSecret
Test-ISCConnection
```

**Store the credential for future sessions:** if SecretManagement/SecretStore is not installed or no default vault exists, guide the user through installing/configuring those modules first:

```powershell
Install-Module Microsoft.PowerShell.SecretManagement -Scope CurrentUser
Install-Module Microsoft.PowerShell.SecretStore -Scope CurrentUser
Register-SecretVault -Name Default -ModuleName Microsoft.PowerShell.SecretStore -DefaultVault
```

Do not disable SecretStore authentication merely to avoid its password prompt. Then store and connect:

```powershell
$clientId = Read-Host 'ISC client ID'
$clientSecret = Read-Host 'ISC client secret' -AsSecureString
New-ISCTenant -Tenant $tenant -Domain $domain -ClientID $clientId -ClientSecret $clientSecret
Connect-ISC -Tenant $tenant -Domain $domain
Test-ISCConnection
```

The stored entry is named `ISC - <tenant> API`. A successful connection is the session's PowerShell tenant for all subsequent cmdlets — but only within that same PowerShell process. If commands run as separate `pwsh` invocations (one per tool call or script file) rather than one ongoing session, each invocation starts with no connection at all: `Connect-ISC` must run again, in the *same* invocation, before any other cmdlet. `Connect-ISC` itself does not accept `-ReconnectAutomatically` — use that switch on the data cmdlets that support it (e.g. `Get-ISCIdentity`, `Invoke-ISCQuery`) to refresh an expired token; do not add it to `Connect-ISC`. Do not print the token returned by `Get-ISCConnection` or `Test-ISCConnection` — both return the same connection object with a `Token` property. If the user switches tenants, ask them to name the new tenant and run `Connect-ISC` again rather than changing the `sail` active environment.

## Step 2a: Establish which `sail` environment to work in

The `sail` CLI persists a single "active" environment across sessions (set via `sail environment use`). Never rely on whatever is left over from a previous session — a person working across multiple ISC tenants (e.g. `sandbox`, `stage`, `prod`) could otherwise have commands silently run against the wrong one.

**Do this once, at the start of every new session, before running any search or api command:**

1. List available environments (reuses the output from Step 1a — no need to re-run):
   ```bash
   printf '\n' | sail environment list 2>&1
   ```
2. Ask the user which environment to work in for this session, presenting the names found above as options. Do not use `sail environment show` to reveal the current active one first — it prints credentials (baseurl/tenanturl aside, `show` includes secrets that `list` does not) and isn't needed here.
3. Once the user picks one, use that name for the **entire session** by appending `--env <name>` to every subsequent `sail search` / `sail api` command. Do not run `sail environment use` to change the persisted default — that mutates global CLI state outside this session and could affect the user's other terminals or scripts. Explicit `--env` on each command is session-scoped and leaves no side effect behind.
4. If the user later asks to switch environments mid-session, just start appending the new `--env <name>` — no need to re-list or re-prompt unless they ask to see the options again.

## Step 3: Determine operation type

> **Safety override — reset and task-status writes:** For any source/account/entitlement reset request, load [docs/investigations/source-reset.md](docs/investigations/source-reset.md) and obtain explicit user confirmation for the exact environment, source, and reset scope (accounts, entitlements, or both) before calling a reset endpoint—even when the user supplied the endpoint or asked for a reset generally. These operations are destructive to ISC data and are normally sandbox/new-source work, not routine production maintenance. For any request to clear, terminate, or force-complete a pending task, load [docs/investigations/stale-task-remediation.md](docs/investigations/stale-task-remediation.md); obtain explicit confirmation for the exact environment, task ID/type, and status change before any task-status PATCH. Never bulk-mark pending tasks as successful.

| Request pattern | Operation |
|---|---|
| Inventory all of a tenant's sources/connectors, find each one's SailPoint connector documentation, or create/refresh profiles for an environment | [Environment connector discovery](docs/connectors/environment-discovery.md); read-only ISC access, local profile writes only after approval. |
| Discover an ISC source/connector's schemas, policies, entitlement types, mappings or capabilities; supply facts for UAT | [Connector configuration discovery](docs/connectors/configuration-discovery.md); read-only by default. |
| Build, create, review or update a connector profile; find functional requirements or design rationale for a known integration | [Profile-building routine](docs/connectors/profile-building.md); load the matching profile from the [connector index](docs/connectors/README.md). |
| Start or resume connector/customizer discovery, interview, decision map or implementation spec | [Discovery → specification](docs/connectors/development/discovery.md); offline-capable; use an issue-tracking skill or process for requested tracker operations when available. |
| Design, build, modify, review or troubleshoot a connector or customizer | [Connector development](docs/connectors/development/README.md); load a matching integration profile and, for generic connectors (Web Services, SCIM, JDBC, Delimited File), the [generic reference](docs/connectors/generic/README.md). |
| "show a specific identity's historical access events" | **api** — `GET /historical-identities/v1/{id}/events` (experimental; see [API reference](docs/api/endpoints.md#historical-identity-events)). |
| "search for identities matching...", "find all accounts...", "search the events index across identities", "look up entitlements..." | **search** |
| Reset a source, its accounts, or its entitlements (including a request naming a reset endpoint) | See [docs/investigations/source-reset.md](docs/investigations/source-reset.md); explicit user confirmation for the exact environment/source is mandatory before any reset call. |
| Investigate or clear an in-progress/stale task blocking a source reset or Identity Profile change | See [docs/investigations/stale-task-remediation.md](docs/investigations/stale-task-remediation.md); inspect read-only first and require explicit task-specific confirmation before any status PATCH. |
| Explicit endpoint path starting with `/`, "call the /accounts/v1 endpoint", "get account activity with ID xyz" (unless it is a reset endpoint; use the reset row above) | **api** |
| Reload or aggregate one specific source account (e.g. "refresh this one Salesforce account") | See [docs/investigations/source-aggregation.md](docs/investigations/source-aggregation.md#single-account-reload); use `POST /accounts/v1/{id}/reload`, not source-wide `load-accounts`. |
| Aggregate a source's accounts (normal or without optimization) or entitlements | See [docs/investigations/source-aggregation.md](docs/investigations/source-aggregation.md) for the three routines, multipart request shape, and task monitoring. |
| Given an external ticket number (ServiceNow RITM/REQ, etc.), "what happened to this access request" | See [docs/investigations/access-request-lookup.md](docs/investigations/access-request-lookup.md) instead — resolves the ticket to an `accessRequestId` first |
| Investigating a provisioning failure or activity for an identity | See [docs/investigations/provisioning.md](docs/investigations/provisioning.md) instead — combines both, in parallel |
| User requests PowerShell, PowerShell objects/pipelines, or an `iscUtils` cmdlet | Use the PowerShell path in [docs/powershell/](docs/powershell/README.md); do not require a `sail` environment for that path |
| Ambiguous — default to **api** if an endpoint path is given, otherwise **search** |

---

## Step 4a: Search operation (`sail search query`)

This section applies only to the `sail` path. For PowerShell, use `Invoke-ISCQuery` or a resource-specific `iscUtils` cmdlet as documented in [docs/powershell/](docs/powershell/README.md).

**Active-people filter:** when identifying currently active people/identities, query `attributes.identityState:ACTIVE`. Do not use the top-level `status:ACTIVE` as a substitute; ISC record status is not the person's current active state. See [docs/search/searchable-fields.md](docs/search/searchable-fields.md#identities-identities-index).

```bash
sail --env <name> search query "<QUERY>" --indices <INDEX> 2>&1
```

Writes results to a file in `search_results/` — read the file back after running. For valid `<INDEX>` values, query-string syntax (wildcards, nested `@object(...)` queries, ranges, `.exact`, `_exists_`, etc.), and searchable fields per index, see [docs/search/](docs/search/README.md).

---

## Step 4b: API operation

For the `sail` path, always use `--pretty` for readable output. For method selection (get/post/put/patch/delete), API versioning, the endpoint list, filter/sort/pagination syntax, and `sail api` flag mechanics (`--query`, `--body`, `-H`, `--jsonpath`), see [docs/api/](docs/api/README.md) and [docs/cli/](docs/cli/README.md).

For the PowerShell path, use the exported `iscUtils` cmdlets and their PowerShell object output as documented in [docs/powershell/](docs/powershell/README.md). Do not mechanically translate a `sail` command into a guessed PowerShell URL: the wrappers use a mixture of API versions and do not cover every endpoint. If the cmdlet does not expose the required operation, explain that limitation and ask before falling back to `sail api`.

---

## Step 5: Handle results

- For `sail`/raw API responses, present JSON as a formatted code block.
- For PowerShell results, preserve the object/pipeline semantics while working; serialize with `ConvertTo-Json -Depth 25` only when JSON is needed for display or a subsequent request.
- For large responses, summarize key fields and offer to show full output.
- If the user asked for a specific ID or field, highlight it.
- For what fields to expect per resource, see [docs/api/response-fields.md](docs/api/response-fields.md).

---

## Environment handling reminder

For the `sail` path, every command in a session must append `--env <name>`, using the environment the user selected in Step 2a — never omit it and never rely on the CLI's persisted default:

```bash
sail --env sandbox api get /accounts/v1 --pretty
sail --env prod search query "name:\"John\"" --indices identities
```

For the PowerShell path, the tenant/domain is established by `Connect-ISC`; subsequent cmdlets use that in-process connection and do not take `--env`. That connection dies with the process — a fresh `pwsh` invocation has no session until `Connect-ISC` runs again inside it. Read [docs/powershell/](docs/powershell/README.md) before connecting, and never print the token exposed by `Get-ISCConnection` or `Test-ISCConnection`.
