# Build or update a connector profile

Use when asked to create, update or review durable institutional knowledge about an integration. Profiles enrich ISC configuration with functional requirements, design justification and operational context; they do not replace live discovery or prove that a test passed. See [configuration discovery](configuration-discovery.md) for a read-only inventory of one source and [environment connector discovery](environment-discovery.md) to draft profiles for every source in a tenant.

## Profile location

Profiles describe one organization's integrations, so they live outside the skill's shipped content unless the user deliberately keeps them there.

1. **Profile root.** Use the folder named in the project's instructions (`AGENTS.md`/`CLAUDE.md`), for example `SailPoint integration profiles: <path>`. If none is named, ask the user once. Suggest a folder in the current project, such as `./sailpoint-profiles/`, and offer to add the line to the project instructions. Do not write inside an installed skill or plugin directory unless the user explicitly points the root there.
2. **Organization folders.** Under the root, keep one folder per organization or client: `<root>/<organization>/`. An organization can own several ISC tenants (production and sandboxes), and one integration profile covers all of them through its environment matrix. Do not split one integration into a copy per tenant.
3. **`tenants.md`.** Each organization folder has a `tenants.md` listing its ISC tenant **subdomains** (from the environment base URL, for example `acme` and `acme-sb`). It can also note a user's local `sail` environment alias for each one. After environment selection, find the folder whose `tenants.md` lists the selected tenant's subdomain. If none does, ask which organization the tenant belongs to before reading or writing profiles. Key on the subdomain, not the alias, because aliases are personal.
4. **Profile files.** Use `<root>/<organization>/<integration>.md`, plus a `README.md` index for that organization's profiles.
5. **Isolation.** Read and write only the selected organization's folder. Do not search, compare against or copy from another organization's profiles, including when looking for related known issues. To share a lesson across organizations, the user must approve a sanitized, connector-general version for an optional `<root>/_shared/` folder. Never copy it automatically.

## Routine

1. **Identify and bound.** Resolve connector/application name and aliases, relevant ISC sources/environments, target-system boundaries and repository. Check the organization's profile folder and its index first (see [Profile location](#profile-location)); update a matching profile instead of making a duplicate. If names, source, environment or scope are ambiguous, ask. Record owner/reviewer if known; do not assume the requester owns the integration.
2. **Gather available evidence.** Inspect the current profile and references, project requirements/discovery, approved decisions, Jira or other work tracking where applicable, repository specification/code/tests, vendor documentation, deployment notes and dated test results. Read the latest issue comments as well as descriptions; do not treat an old description as current merely because it is an issue field. Record the repository branch/commit and distinguish it from deployed version. If needed and authorized, run read-only [configuration discovery](configuration-discovery.md); offline work does not require a live connection.
3. **Reconcile by fact.** Mark whether a claim is an approved requirement, intended design, repository implementation, configured/live observation, dated test observation, inference or unknown. Cite the source, relevant environment/version and date. Requirements answer what was agreed; source configuration answers what is configured; code answers what that revision implements; a test or log answers only what it actually demonstrated. None automatically overrides the others. Surface contradictory or missing evidence, including ambiguous requirements, instead of resolving it by assumption.
4. **Draft or propose an update** using the structure below. Retain useful decision history and profile facts; do not overwrite an existing profile wholesale without reviewing its content. Preserve an unresolved conflict until an owner resolves it. Use stable links and concise summaries; avoid duplicating rapidly changing issue status or copying large specifications into the profile.
5. **Review and publish.** Show the draft and key gaps to the requester/owner; ask for corrections and approval before writing a new profile or consequential changes unless the user has already authorized the specific edit. Save under `<root>/<organization>/<integration>.md` and add/update that organization's index. Keep unapproved profiles explicitly labeled draft. Updating a profile does not authorize ISC or Jira changes.
6. **Maintain.** Refresh affected sections after approved requirements, source configuration, implementation or operational findings change. Record what was checked, when and in which environment/version, and link the evidence. Never update a historical test to imply current behavior; retain prior evidence and note supersession.

## Profile structure

Use headings appropriate to the integration; mark missing information **unknown** and omit irrelevant empty sections only with an explanation where coverage matters.

- **Status and identification:** draft/approved, application/connector aliases, profile owner/reviewer when known, relevant environments/sources, repository and last evidence review date. Distinguish tenant-specific facts and avoid disclosing sensitive identifiers unnecessarily.
- **Sources by tenant:** for each tenant subdomain, the source name and ID, `connectorName`/`connector`, and SaaS or VA (cluster). Record when it was checked.
- **Connector documentation:** the matched SailPoint guide and its `llms.txt` (from [environment connector discovery](environment-discovery.md)), SaaS or VA, the reviewed date, and a short **read before acting** list: deprecations, defaults with large effects, safety warnings and conflicts between vendor pages, each with a page link. Link the vendor pages; do not copy their content. For generic connectors, also link the matching [generic reference](generic/README.md). Custom connectors link their repository instead.
- **Purpose, functional requirements and exclusions:** approved outcomes, target populations and governed access; separately state proposed or unresolved requirements.
- **Architecture and data flow:** target systems and APIs, source boundaries, account/entitlement types, authoritative sources, mappings and identifiers/correlation.
- **Operations and behavior:** read/list, create, update, per-type entitlement add/remove, sync and lifecycle as applicable. Separate implementation, actual source configuration, supported use and verification.
- **Design decisions and rationale:** alternatives, why a design was chosen, constraints, evidence and date. Label a plausible explanation as an inference until an owner confirms it.
- **Environment/deployment matrix:** environment/source, repository revision and deployed version/tag if verified, dates and evidence. Historical observations are not a current inventory.
- **Risks, limitations and operations:** pagination, ambiguity, security, failure/retry behavior, logs, vendor limitations and safe troubleshooting. Label source-specific findings.
- **Known issues:** the troubleshooting pattern catalog for this integration (see [Known issues](#known-issues)).
- **Open questions and conflicts:** owner/decision and next evidence needed when known. Include missing live configuration and approvals.
- **Source map and verification history:** stable paths/links, ticket comments or evidence identifiers, dates, review status and any superseding decisions. Link UAT artifacts; do not copy their mutable execution status into a permanent claim.

## Known issues

The profile's known-issues table records error signatures and symptoms found during troubleshooting. These are usually things the vendor documentation does not cover, or that look different in ISC than the vendor describes. Do not copy rows from the vendor troubleshooting page; link it from the connector documentation section instead. Add a row only when it adds something the vendor page does not, such as how the error appears in ISC account activity compared with the connector log.

| # | Scope | Match on (exact error text / symptom) | Cause | Resolution | Evidence |
|---|---|---|---|---|---|
| 1 | connector-general / integration-specific / unverified | `<error text>` | <cause> | <resolution and whether it self-resolves> | <tenant, date, how it was confirmed> |

- **Scope** separates connector behavior from this integration's configuration, customizer, target application or data. **Connector-general** means another organization using the same connector would hit it; confirm that through vendor documentation, connector source code or a second tenant. Record an issue as **unverified** until its scope is established.
- **Related profiles.** When troubleshooting one integration, also scan the known issues of the same organization's other profiles that use the same connector for connector-general and unverified rows. Stay within the organization folder.
- **Adding a row.** When a troubleshooting session confirms a pattern that is not already listed, propose the row with its evidence (tenant, date, what was traced) and the profile it belongs in. Write it only after the user approves it. A hypothesis that was not traced to a cause is not a known issue.
- **Maintenance.** Keep superseded rows and mark them, so that later readers can see that something used to happen.

## Security and sharing

Do not read or copy local credential files, secrets, refresh/access tokens, authorization headers, raw source connector configuration or unnecessary PII into profiles, issues or logs. DEBUG logs can contain identity data even if credentials are masked. Before publishing internal design or operational detail, check the repository/plugin's distribution and intended readership; if access is uncertain, mark publication/review as pending and avoid sensitive content. Use restricted evidence locations and redacted links where appropriate.

## Quality check

Before proposing the edit, verify every referenced local file/link exists, each important claim has a source, time-bound observations have a date/environment/version, inferred and unknown behavior are labeled, contradictions are surfaced, and the profile has no secrets. A profile may legitimately remain partial or draft; do not invent approval, ownership or a current deployed version to make it complete.
