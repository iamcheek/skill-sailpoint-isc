# PowerShell and `iscUtils`

`iscUtils` is an optional PowerShell execution path for Identity Security Cloud (ISC) API work. It is published in the PowerShell Gallery; users do not need a local checkout of its source code. Use it when the user explicitly asks for PowerShell, wants PowerShell pipeline/object output, or a high-level cmdlet is a better fit than composing a raw request. Use `sail api` for arbitrary endpoints, current endpoint/version behavior, or when a wrapper does not expose the needed option.

## Tool selection

The orchestrating skill asks the user at the beginning of the session whether to use `sail` or try `iscUtils` PowerShell first. If PowerShell-first is selected, prefer the module for the session and ask before falling back to `sail` when PowerShell is unavailable or a wrapper is missing. Choose one execution path for a session and keep its authentication context consistent:

| Situation | Use |
|---|---|
| Arbitrary endpoint, exact request body, current per-resource API path, or clean scriptable JSON | `sail api` |
| Identity/account/entitlement/access-profile/role lookup with PowerShell objects and pipelines | `iscUtils` cmdlets |
| Workflow, workflow-execution, transform, source/schema, task, or connector-rule lookup | `iscUtils` cmdlets when their parameters fit; otherwise `sail api` |
| Platform-log lookup with a time window and filters | The documented `/platform-logs/v1/query` API path via `sail api`; use a PowerShell wrapper only if the installed `iscUtils` version exports one |
| A write operation | Either path, but inspect first and obtain confirmation immediately before the mutation unless the user explicitly authorized it |

The cmdlets are wrappers, not a complete replacement for the API. The current module uses a mixture of `/v3`, `/v2024`, `/v2025`, `/v2026`, and `/beta` paths. Do not infer that a wrapper's path is the current canonical path; check [the API reference](../api/README.md) when version currency matters.

## Select the tenant for the session

Unlike the `sail` CLI, `iscUtils` does not provide a tenant-list command or a persisted active tenant. At the start of a PowerShell session, ask the user which tenant subdomain and domain to use (`Default`, `Demo`, or `FedRamp`). Do not infer it from a previous `sail` environment or PowerShell process.

After the user names a tenant, check its expected stored-credential entry without retrieving or printing the credential:

```powershell
$tenant = '<tenant-subdomain>'
$domain = '<Default|Demo|FedRamp>'
$storedCredential = if (Get-Command Get-SecretInfo -ErrorAction SilentlyContinue) {
    Get-SecretInfo -Name "ISC - $tenant API" -ErrorAction SilentlyContinue
}
$storedCredential -ne $null
```

If it exists, connect with `Connect-ISC`. If it does not, use either the runtime-only or stored-credential setup below. A successful `Connect-ISC` establishes the tenant for subsequent cmdlets in that PowerShell process.

> **Process persistence.** `iscUtils` state — the token and an internal source-name cache that `-Source` parameters validate against — lives only in the current PowerShell process. If commands run as separate `pwsh` invocations (one per tool call or script file) rather than one ongoing session, each invocation starts with no connection at all: `Connect-ISC` must run again, in the *same* invocation, before any other cmdlet. Calling `Get-ISCSource`, `Get-ISCAccount -Source`, `Get-ISCEntitlement -Source`, `Get-ISCSourceSchema`, or `Invoke-ISCAccountAggregation` before `Connect-ISC` has run in that process fails with a misleading first error — `New-Object: ... ValidateSetAttribute ... constructor not found` — followed by a clearer second line, `Connection to Identity Security Cloud has not been established`. The first line is not a module bug; it means `Connect-ISC` needs to run in this same invocation, not an earlier one.

## Prerequisites, installation, and import

The module requires PowerShell 7.2 or later. On macOS/Linux, use `pwsh`, not Windows PowerShell 5.1. Install it from PSGallery only when it is not already available:

```powershell
if (-not (Get-Module -ListAvailable -Name iscUtils)) {
    Install-Module -Name iscUtils -Repository PSGallery -Scope CurrentUser
}

Import-Module iscUtils -ErrorAction Stop
Get-Module iscUtils
Get-Command -Module iscUtils
```

Do not use `-Force` or update an existing installation automatically. If a newer version is needed, check it explicitly with `Find-Module -Name iscUtils -Repository PSGallery` and obtain approval before updating. A local module source checkout is useful for module development, but must not be a runtime prerequisite for this skill. The published module can lag a source checkout, so verify that a requested cmdlet exists with `Get-Command -Module iscUtils` before using it.

## Authentication

`iscUtils` uses client-credentials PATs and keeps the access token in the current PowerShell process. The tenant argument is the tenant subdomain only (for example, `mycompany`), not a full URL.

### Runtime credentials

Prefer runtime credentials when a SecretManagement vault is not already configured. Read the secret interactively so it is not placed in the command line or transcript:

```powershell
$clientSecret = Read-Host 'ISC client secret' -AsSecureString
Connect-ISC -Tenant '<tenant>' -Domain Default `
  -ClientID '<client-id>' -ClientSecret $clientSecret
Test-ISCConnection
```

Valid `-Domain` values are `Default` (`*.api.identitynow.com`), `Demo` (`*.api.identitynow-demo.com`), and `FedRamp` (`*.api.saas.sailpointfedramp.com`).

### Stored credentials

The module can retrieve a credential named `ISC - <tenant> API` from a PowerShell SecretManagement vault. This requires the SecretManagement/SecretStore modules and a configured vault:

```powershell
New-ISCTenant -Tenant '<tenant>' -Domain Default
Connect-ISC -Tenant '<tenant>'
```

Do not print the credential or the access token. `Get-ISCConnection` and `Test-ISCConnection` both return the same connection object, which includes a `Token` property — select only safe connection metadata when checking either, and don't call either one bare (unassigned/unpiped) if the output will be displayed:

```powershell
Get-ISCConnection | Select-Object Tenant, Domain, 'API URL', Timestamp, TokenExpiration
Test-ISCConnection | Select-Object Tenant, Domain, 'API URL', Timestamp, TokenExpiration
```

Many data cmdlets (e.g. `Get-ISCIdentity`, `Get-ISCAccount`, `Invoke-ISCQuery`) accept `-ReconnectAutomatically` to refresh an expired token using the previous connection's tenant/domain. `Connect-ISC` itself does **not** take this parameter (confirmed against the installed v1.10 package's `param()` block) — it always requires an explicit `-Tenant`/`-Domain`, and credentials if no stored connection exists yet in this process.

## Common cmdlet mappings

| Task | Cmdlet and useful parameters | Wrapper path(s) used by the current module |
|---|---|---|
| Search supported ISC indices | `Invoke-ISCQuery -Query '<Lucene query>' -Index Identities` | `/v3/search` |
| Identity lookup/list | `Get-ISCIdentity -ID`, `-EmployeeNumber`, `-CustomQuery`, `-List`, `-Active`, `-IncludeNested` | `/v3/search` (`identities`) |
| Account lookup/list | `Get-ISCAccount -ID`, `-IdentityID`, `-List`, `-Source`, `-Uncorrelated`, `-SchemaAttributes` | `/v3/accounts` |
| Entitlement lookup/list | `Get-ISCEntitlement -ID`, `-AccountID`, `-Name`, `-Value`, `-Source`, `-Requestable`, `-NotRequestable`, `-OwnerId`, `-List` | `/beta/entitlements` |
| Access-profile lookup/list | `Get-ISCAccessProfile -ID`, `-Name`, `-List` | `/v3/access-profiles` |
| Role lookup/list | `Get-ISCRole -ID`, `-Name`, `-List`, `-Requestable`, `-NotRequestable`, `-OwnerId` | `/v2025/roles` |
| Source lookup/list | `Get-ISCSource -Source`, `-List` | `/v3/sources` |
| Source schemas | `Get-ISCSourceSchema -Source -SchemaName`, `-SchemaId`, or `-List` | `/v3/sources/{sourceId}/schemas` |
| Transforms | `Get-ISCTransform -Name` or `-All` | `/v3/transforms` |
| Workflows | `Get-ISCWorkflow -ID`, `-All`, `-Enabled`, or `-Disabled` | `/v3/workflows` |
| Workflow execution history | `Get-ISCWorkflowExecution -ID` | `/v3/workflow-executions/{id}/history` |
| Workflow execution list | `Get-ISCWorkflowExecutionList -ID -StartedAfter -StartedBefore -Status` | `/v3/workflows/{id}/executions` |
| Connector rules | `Get-ISCConnectorRule -Id` or `-All` | `/beta/connector-rules` |
| Identity attributes | `Get-ISCIdentityAttribute -List`; `Get-ISCIdentityAttributeList` | `/beta/identity-attributes` or `/v2024/identity-attributes` |
| Task status | `Get-ISCTaskList`, `Get-ISCPendingTaskList` | `/beta/task-status` |
| Platform logs | Not exported by the currently published PSGallery package; use `sail api` for `/platform-logs/v1/query` unless `Get-Command -Module iscUtils` shows a wrapper in a later installed version | N/A |

Examples:

```powershell
# Search identities; the result is PowerShell objects, not a mixed CLI log/JSON stream.
Invoke-ISCQuery -Query 'attributes.lastname:ackbar' -Index Identities

# Retrieve one identity and its correlated accounts.
$identity = Get-ISCIdentity -EmployeeNumber '12345' -IncludeNested
$identity.accounts

# List uncorrelated accounts from a source.
Get-ISCAccount -List -Source 'Active Directory' -Uncorrelated

# Platform logs are not exported by the currently published PSGallery package;
# use the documented sail api path for that operation.

# Serialize objects only when JSON is required by the next step.
Get-ISCIdentity -ID '<identity-id>' | ConvertTo-Json -Depth 25
```

`Invoke-ISCQuery` supports the module's `AccessProfiles`, `AccountActivities`, `Entitlements`, `Events`, `Identities`, `Roles`, and `*` index values. For a provisioning investigation, prefer the playbook's account-activity and platform-log correlation steps; do not assume the wrapper's pagination behavior is identical to the current API reference.

> **Published-package caveat:** the current PSGallery package does not export `Invoke-ISCPlatformLogSearch`, even though a newer source checkout may contain it. Always inspect the installed module's exported commands before selecting a wrapper.

> **Current-package caveat:** source review found that `Get-ISCIdentityAttribute -Name` does not initialize the request URI in the current published package, so use `-List`, `Get-ISCIdentityAttributeList`, or the documented API path instead of relying on the name form until the module is fixed.

> **Current-package caveat:** `Invoke-ISCQuery` hardcodes `includeNested = $false` in its request body and exposes no `-IncludeNested` parameter. This does not stop nested boolean filters from matching — `@accounts(source.name:"Okta" AND disabled:true)` correctly filtered live identities (verified by spot-checking sample results directly against `Get-ISCIdentity -IncludeNested`) — but the identity objects `Invoke-ISCQuery` returns will not carry full nested account/access detail. Follow up with `Get-ISCIdentity -ID <id> -IncludeNested` when the matching account's own fields (not just the identity's) are needed.

> **Quoting Lucene queries from Bash.** `\"` is not a PowerShell string escape — embedding a double-quoted Lucene literal inside a `pwsh -Command '...'` string with `\"` will fail to parse. Use a single-quoted PowerShell string for the query instead (`'@accounts(source.name:"Okta" AND disabled:true)'`), or write the script to a temp `.ps1` file and invoke with `-File` when the query has several embedded quotes.

## Writes and operational actions

The module also exports mutating or operational cmdlets:

- `New-ISCRole`, `Set-ISCRole`
- `Set-ISCAccessProfile`, `Set-ISCEntitlement`
- `Set-ISCIdentityLifecycleState`
- `Set-ISCTaskCompleted`
- `Invoke-ISCAccountAggregation`
- `New-ISCTenant`, `Remove-ISCTenant`

Before invoking one, identify the target and show the intended change. For lifecycle state changes, the wrapper first resolves the identity profile's lifecycle states; verify the exact state name and target identity first. Treat `Remove-ISCTenant` as a credential-store deletion, not an ISC tenant deletion.

The module's public functions can evolve independently of this skill. If a cmdlet returns an unexpected result, fails because an endpoint/version changed, or lacks a needed parameter:

1. Capture the cmdlet error without exposing credentials.
2. Check the current endpoint and required headers in `docs/api/`.
3. Fall back to `sail api` for the same operation when appropriate; do not guess a replacement PowerShell URL.
