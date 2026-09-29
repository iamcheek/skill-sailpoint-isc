---
name: sailpoint-isc
description: >
  Query SailPoint Identity Security Cloud using the sail CLI. Handles natural-language requests
  to search identities, account activities, entitlements, events, access profiles, or roles via
  `sail search query`; direct ISC REST API calls via `sail api get/post/patch/put/delete`; and
  transform/workflow/rule management via `sail transform`/`sail workflow`/`sail rule` (list, preview,
  download, create, update, delete), plus the Transform JSON reference (docs/transform/).
  Use when the user asks to look up an identity, pull account activity, search for entitlements,
  call a specific ISC API endpoint (e.g. "/accounts/v1", "/account-activities/v1"), or work with
  a transform/workflow/rule definition. Do NOT use for sail commands outside this scope (e.g.
  connectors, clusters, virtual appliances, SPConfig, report generation).
user-invocable: true
disable-model-invocation: false
argument-hint: "<natural language request or API endpoint>"
allowed-tools: Bash(sail *) Bash(jq *) Bash(echo *) Bash(printf *) Bash(cat *) Bash(grep *)
compatibility: "Requires the SailPoint sail CLI."
---

# SailPoint ISC

Interact with SailPoint Identity Security Cloud through the SailPoint `sail` CLI:

1. **`sail search query`** — full-text Lucene search across ISC indices
2. **`sail api`** — direct REST calls to any ISC endpoint
3. **`sail transform` / `sail workflow` / `sail rule`** — dedicated subcommands for managing those three resource types (list, preview, download, create, update, delete)

This file covers session setup and routing only. Deep reference material lives in `docs/` — load the relevant file only when the task needs it. Use the CLI path for every ISC operation:

| Directory | Covers |
|---|---|
| [docs/cli/](docs/cli/README.md) | `sail` CLI flag mechanics, output capture, CLI-level errors |
| [docs/api/](docs/api/README.md) | REST API versioning, endpoints, filter/sort/pagination syntax, response fields |
| [docs/search/](docs/search/README.md) | Search indices, query-string syntax, audit event search & reference |
| [docs/transform/](docs/transform/README.md) | Transform JSON structure, every transform operation, provisioning-policy transforms, and `sail transform` usage |
| [docs/roles/](docs/roles/README.md) | Roles API read/update/delete operations, JSON Patch membership criteria, role-management permissions, and post-update verification |
| [docs/workflows/](docs/workflows/README.md) | Date-windowed workflow execution searches, `history-v2` traversal, runtime trigger-input lookup, and workflow/event cross-checks |
| [docs/investigations/](docs/investigations/README.md) | Task-specific playbooks (e.g. provisioning failure investigation, ticket-number-to-access-request lookup) |
| [docs/connectors/](docs/connectors/README.md) | Per-connector troubleshooting reference (e.g. Salesforce SaaS known issues) — load a connector's file once it's identified as the source involved |
| [docs/access-intelligence-center/](docs/access-intelligence-center/README.md) | Access Intelligence Center (Qlik-based BI/reporting) — dashboards, sheets, and the underlying reporting data model. No `sail`/API surface; UI- and data-model-only reference. |

> **`sail transform`/`sail workflow`/`sail rule` vs. `sail api`:** for `preview`, `download`, and batch `create`/`delete` on these three resources, use the dedicated subcommand — it does more than raw CRUD (e.g. `transform preview` evaluates a transform against live identity data; `workflow download`/`rule download` bulk-export) and has no clean `sail api` equivalent. For anything the subcommand doesn't cover (e.g. reading a single transform by ID with `--jsonpath`), fall back to `sail api` against the same resource. See [docs/transform/README.md](docs/transform/README.md#cli-usage) for the transform-specific version of this split.

If the `sail` path is not installed or no environment is configured, use the `setup-sailpoint-cli` skill instead of troubleshooting manually.

---

## Step 1: Check the `sail` path is available

> **Note:** always pipe a newline in — running this bare blocks forever. See [docs/cli/README.md#environment-listing](docs/cli/README.md#environment-listing) for why.

```bash
printf '\n' | sail environment list 2>&1
```

If "command not found", or if `sail environment list` returns no environments, use the `setup-sailpoint-cli` skill to install the CLI and/or walk the user through creating an environment and authenticating, then return here.

## Step 2: Establish which `sail` environment to work in

The `sail` CLI persists a single "active" environment across sessions (set via `sail environment use`). Never rely on whatever is left over from a previous session — a person working across multiple ISC tenants (e.g. `sandbox`, `stage`, `prod`) could otherwise have commands silently run against the wrong one.

**Do this once, at the start of every new session, before running any search or api command:**

1. List available environments (reuses the output from Step 1 — no need to re-run):
   ```bash
   printf '\n' | sail environment list 2>&1
   ```
2. Ask the user which environment to work in for this session, presenting the names found above as options. Do not use `sail environment show` to reveal the current active one first — it prints credentials (baseurl/tenanturl aside, `show` includes secrets that `list` does not) and isn't needed here.
3. Once the user picks one, use that name for the **entire session** by appending `--env <name>` to every subsequent `sail search` / `sail api` command. Do not run `sail environment use` to change the persisted default — that mutates global CLI state outside this session and could affect the user's other terminals or scripts. Explicit `--env` on each command is session-scoped and leaves no side effect behind.
4. If the user later asks to switch environments mid-session, just start appending the new `--env <name>` — no need to re-list or re-prompt unless they ask to see the options again.

## Step 3: Determine operation type

| Request pattern | Operation |
|---|---|
| "search for identities matching...", "find all accounts...", "show events for...", "look up entitlements..." | **search** |
| Explicit endpoint path starting with `/`, "call the /accounts/v1 endpoint", "get account activity with ID xyz" | **api** |
| Given an external ticket number (ServiceNow RITM/REQ, etc.), "what happened to this access request" | See [docs/investigations/access-request-lookup.md](docs/investigations/access-request-lookup.md) instead — resolves the ticket to an `accessRequestId` first |
| Investigating a provisioning failure or activity for an identity | See [docs/investigations/provisioning.md](docs/investigations/provisioning.md) instead — combines both, in parallel |
| Ambiguous — default to **api** if an endpoint path is given, otherwise **search** |

---

## Step 4a: Search operation (`sail search query`)

Use the `sail` CLI search path for this operation.

```bash
sail --env <name> search query "<QUERY>" --indices <INDEX> 2>&1
```

Writes results to a file in `search_results/` — read the file back after running. For valid `<INDEX>` values, query-string syntax (wildcards, nested `@object(...)` queries, ranges, `.exact`, `_exists_`, etc.), and searchable fields per index, see [docs/search/](docs/search/README.md).

---

## Step 4b: API operation

For API operations, always use `--pretty` for readable output. For method selection (get/post/put/patch/delete), API versioning, the endpoint list, filter/sort/pagination syntax, and `sail api` flag mechanics (`--query`, `--body`, `-H`, `--jsonpath`), see [docs/api/](docs/api/README.md) and [docs/cli/](docs/cli/README.md).

---

## Step 5: Handle results

- For `sail`/raw API responses, present JSON as a formatted code block.
- For large responses, summarize key fields and offer to show full output.
- If the user asked for a specific ID or field, highlight it.
- For what fields to expect per resource, see [docs/api/response-fields.md](docs/api/response-fields.md).

---

## Environment handling reminder

Every command in a session must append `--env <name>`, using the environment the user selected in Step 2 — never omit it and never rely on the CLI's persisted default:

```bash
sail --env sandbox api get /accounts/v1 --pretty
sail --env prod search query "name:\"John\"" --indices identities
```
