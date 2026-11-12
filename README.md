# SailPoint ISC Skill

A standalone agent skill for working with SailPoint Identity Security Cloud (ISC) through the `sail` CLI or the optional `iscUtils` PowerShell module.

The skill supports searching identities, accounts, entitlements, access profiles, roles, events, and account activities; making direct ISC REST API calls; investigating provisioning failures and access requests; running gated source aggregation, reset, and stale-task routines; managing transforms, workflows, roles, and rules; planning Configuration Hub migrations; and discovering, designing, building, and troubleshooting connectors and customizers. Offline design and repository analysis work without a CLI or tenant connection.

## Capabilities

- **Search:** Use `sail search query` across ISC search indices.
- **Direct API access:** Use `sail api get`, `post`, `put`, `patch`, and `delete` with ISC-specific endpoint, filtering, and pagination guidance.
- **PowerShell access:** Optionally use `iscUtils` cmdlets for PowerShell objects and pipelines.
- **Provisioning investigations:** Correlate account activities, platform logs, and account state.
- **Access-request lookup:** Resolve external ticket metadata to an ISC access request and verify the resulting access directly.
- **Source operations:** Run account/entitlement aggregation, single-account reloads, source resets, and stale-task remediation, with explicit confirmation required before destructive or task-status writes.
- **Transform, workflow, role, and rule management:** Use the dedicated `sail` subcommands and Roles API guidance with the corresponding JSON references.
- **Configuration Hub:** Plan scoped backups, drafts, mappings, approvals, and deployments between tenants.
- **Connector engineering:** Tenant-wide connector discovery that matches every source to its SailPoint connector guide and drafts integration profiles; read-only source configuration discovery; integration profiles with a known-issues catalog that grows during troubleshooting; generic connector references (Web Services SaaS so far); SaaS connector/customizer discovery-to-specification, SDK and command references.

## Prerequisites

- The `sail` CLI when using the CLI path.
- PowerShell 7.2+ and the optional `iscUtils` module when using the PowerShell path.
- An authenticated ISC environment or tenant connection, depending on the selected path.
- None of the above for offline design, profile drafting, or local repository analysis.

If the `sail` CLI is missing or no environment is configured, use the separately available `setup-sailpoint-cli` skill for installation and interactive environment/authentication setup.

## Usage

Load `SKILL.md` as the skill entry point. At the start of an ISC session it asks whether to use the `sail` CLI or PowerShell `iscUtils` first, then establishes the appropriate environment or tenant context before querying ISC.

Example requests include:

- Search for identities with a particular attribute.
- Show recent account activity for an identity.
- Call an ISC endpoint such as `/accounts/v1` with filters.
- Investigate why a provisioning operation failed.
- Resolve an external request number to its access-request status.
- Build or troubleshoot a SailPoint transform.
- Find the SailPoint documentation for every connector in a tenant and draft a profile for each integration.
- Inventory a source's schemas, provisioning policies, and attribute-sync mappings before a connector change.
- Start discovery for a new SaaS connector or customizer and draft an implementation spec.

## Project structure

```text
.
├── SKILL.md                 # Skill entry point and request-routing workflow
├── AGENTS.md                # Agent-session instructions
├── CLAUDE.md                # Loads AGENTS.md for Claude-compatible sessions
└── docs/                    # On-demand reference material
    ├── access-intelligence-center/
    ├── api/
    ├── cli/
    ├── configuration-hub/
    ├── connectors/
    │   ├── development/     # SaaS connector/customizer discovery, SDK, commands, testing
    │   └── generic/         # Generic connector references (Web Services, SCIM, JDBC, ...)
    ├── investigations/
    ├── powershell/
    ├── roles/
    ├── search/
    ├── transform/
    └── workflows/
```

The entry point intentionally stays focused on session setup and routing. Load deeper documentation only when the request requires it.

## Contributing

Update the relevant reference under `docs/` when ISC API behavior, CLI behavior, or investigation methodology changes. Keep tenant-specific identifiers, credentials, exports, organization-specific integration profiles, and other environment-specific data out of this repository. Integration profiles belong under your own profile root (`profiles/` and `sailpoint-profiles/` are gitignored here).
