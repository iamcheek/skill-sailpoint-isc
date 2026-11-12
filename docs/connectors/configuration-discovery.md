# Connector configuration discovery

Use this read-only routine to inventory an ISC source for connector design, build, troubleshooting, profiling or UAT planning. It produces evidence-backed facts, not a design approval or a test result. For the overall connector index see [README.md](README.md); to turn facts and business context into a durable profile see [profile-building.md](profile-building.md).

## 1. Establish scope and safe access

1. Resolve the application/connector, exact ISC tenant and source ID/name, and target tenant/workspace. If names are ambiguous, show safe identifying metadata and ask which source; do not pick the first match. Record retrieval time, applicable environment, and version/tag only when independently known.
2. If offline artifacts suffice, inspect the repository, `connector-spec.json` or equivalent, existing connector profile and requirements. Label these as repository or historical evidence, not live ISC configuration. Do not force CLI installation/authentication for offline work.
3. If live access is needed, use the parent skill's execution-path and tenant-selection flow before the first call. Respect the user's selected `sail` or PowerShell path and fallback rules. Discovery is read-only: **do not** start aggregation, Test Connection, provisioning, reset, sync, credential rotation or deployment as a discovery step.
4. Source/config responses, policies and logs may contain secrets or personal data, including fields not named `secret`. Do not print or persist full connector configuration, credentials, tokens or unnecessary account data. Prefer narrow projections; sanitize before displaying or storing anything. If an endpoint cannot be safely projected, use an approved secure capture/redaction method or leave the field unknown. Respect least privilege and avoid protected debug output.

## 2. Inventory and reconcile

For each category, record the value, source/artifact and retrieval date, environment/version, and status: **observed**, **configured**, **implemented**, **required/approved**, **inferred**, or **unknown** as appropriate. One fact may have multiple statuses when supported by separate evidence. A 403, 404, empty result, inaccessible UI or missing repository file is not proof that a feature is unsupported; explain the gap and reason.

| Category | Discover and distinguish |
|---|---|
| Source and connector | Source identity/type, connector family, applicable target/workspace, configuration and deployment/version when accessible. Distinguish repository branch/commit, deployed connector version and source configuration. |
| Account schema | Identity/native keys, display and account attributes, data types, multivalue fields, target-to-ISC mapping, correlation rule, transforms, status and null/default behavior where available. Separate repository schema from the live source. |
| Entitlement schemas | Every defined schema/type, identity/display attribute and account association; distinguish a type from an individual entitlement value and the application's actual access object. Compare configured schema with observed catalog separately. A schema named `group` need not represent a target-system group. |
| Create policy | Actual source provisioning policy, required/optional fields, transformations, defaults and input source. The presence of a create command or sample template does not prove what is configured in the UI. |
| Attribute sync | Actual enabled mappings and triggering path, identity source attributes, target fields and transformations. Connector-supported writable fields are not necessarily configured or enabled for sync. |
| Operations | Read/list, create/update, per-type add/remove, enable/disable/delete and other operations as applicable. Distinguish advertised capability, configured path, implemented handler, approved scope, and tested behavior. A schema or role catalog does not establish provisioning support. |
| Security/operations | Auth method and permission names only, not secret values; endpoints or configuration identifiers only when safe to disclose; known scheduling, diagnostics and deployment/version data. |
| Risks and scope | Exclusions, correlation/join behavior, pagination and status semantics, eventual consistency, partial writes, approval dependencies and conflicts across requirements/configuration/code. |

Use complete pagination and explicitly note filters and coverage when listing resources. Do not compare raw counts across different populations or infer supported entitlement types from the values currently returned alone.

## 3. Read paths available in existing references

Use a narrow query or projection and the selected environment; verify endpoint availability, permissions and relevant API version rather than guessing. Examples of documented read surfaces:

- Source metadata: `/sources/v1` and `/sources/v1/{id}`. Source detail may contain connector configuration: **do not display the raw response**. The safe-name projection below is illustrative, not a complete discovery:

  ```sh
  sail --env <chosen-env> api get /sources/v1/<source-id> --jsonpath '$.name' --pretty
  ```

- Connector version/tag: for a custom SaaS connector, resolve the **platform connector ID** separately from the source's connector/tag/spec ID; do not assume those identifiers are interchangeable. With the source's alias and environment confirmed, read `/beta/platform-connectors`, safely filter its returned `id`/`alias` pairs to the matching alias, and use `sail --env <chosen-env> conn tags list -c <platform-connector-id>` to inspect tag names and active versions. Match the source's configured tag before reporting a version. In one live check, a source's connector ID matched its `latest` **tag ID**, and using that value as the platform connector ID returned 404; resolving the alias to the platform ID and listing its tags returned the active version. This is a source-specific observation, not a universal ID mapping. A tag version is not proof of a repository commit/bundle match.
- Source schemas: `GET /sources/v1/{sourceId}/schemas` lists configured schemas. The `iscUtils` mapping documents `Get-ISCSourceSchema` and its v3 wrapper path; see [PowerShell mappings](../powershell/README.md#common-cmdlet-mappings). Honor the selected execution path and module availability. In a `sail` session this narrow projection lists names only, not the whole schema:

  ```sh
  sail --env <chosen-env> api get /sources/v1/<source-id>/schemas --jsonpath '$[*].name' --pretty
  ```

  Apply pagination when the chosen endpoint supports/requires it; inspect identifiers, attributes and associations through safe projections or an approved sanitized capture. Schema names alone are not sufficient for a coverage matrix.
- Provisioning policies: `GET /sources/v1/{sourceId}/provisioning-policies/{usageType}` retrieves one policy. Usage types and the existing v3 equivalents are described in [provisioning policies](../transform/provisioning-policies.md#provisioning-policy-api). For example, inspect CREATE field names without printing policy values:

  ```sh
  sail --env <chosen-env> api get /sources/v1/<source-id>/provisioning-policies/CREATE --jsonpath '$.fields[*].name' --pretty
  ```

  Fetch only applicable policies and sanitize values before output. Field names alone do not show mapping/transform definitions, and a policy by itself does not establish an enabled attribute-sync mapping.
- Attribute sync: `GET /sources/v1/{id}/attribute-sync-config` is experimental and requires `X-SailPoint-Experimental: true`. The current spec describes user authentication, `idn:attr-sync-source-config:read` or manage, and ORG_ADMIN or HELPDESK. Its `attributes` array includes **both enabled and disabled** mappings; each entry has identity attribute `name`, account attribute `target`, and `enabled`:

  ```sh
  sail --env <chosen-env> api get /sources/v1/<source-id>/attribute-sync-config -H 'X-SailPoint-Experimental: true' --jsonpath '$.attributes' --pretty
  ```

  Report enabled and disabled mappings separately and protect sensitive configuration. The documented v1 route and permission contract were checked against the SailPoint Sources API specification; a GET with a sanitized projection succeeded in live testing. The exact example projection above has not been separately live-tested. A 403 or unsupported route means unknown, not no mappings. Do not call the write method or trigger sync during discovery.
- Correlation: `GET /sources/v1/{id}/correlation-config` returns correlation configuration. Inspect `attributeAssignments` securely and sanitize potential literal/filter values before displaying. This is ISC account-to-identity correlation, which can differ from an internal connector join. A missing/empty configuration or unavailable rule needs interpretation; do not infer how all accounts correlate.
- Existing entitlement objects: [endpoint reference](../api/endpoints.md) and [filtering/pagination](../api/filtering-sorting-pagination.md) distinguish source filters and the v1 versus beta data paths. Catalog entries are not a substitute for schema discovery or proof of write support.

The per-resource routes above were checked against SailPoint's [`api-specs`](https://github.com/sailpoint-oss/api-specs) Sources OpenAPI at commit `d0d46de7` (2026-07-08), including the schema, policy, sync and correlation GET contracts. Sanitized GETs for all five surfaces succeeded in live testing against one source; the exact JSONPath examples above have not been separately live-tested, and other tenants/permissions can differ. Verify current published documentation and deployed support. The references do not establish a complete read route for every deployed version, rule-based correlation detail or connector operation. Use current documentation, authorized UI inspection, the source repository or operator-provided configuration; if unavailable, record **unknown** rather than inventing an endpoint or claiming no configuration exists. Do not expose raw response bodies when diagnosing an error.

## 4. Handoff

Return a concise inventory with:

- Exact source/environment and evidence timestamp; repository commit and deployed version separately if known.
- Account schema/keys and mapping summary; each entitlement type, account association, and confirmed versus unknown read/write capabilities.
- Create policy and every confirmed sync mapping; applicable lifecycle operations and scope.
- Evidence references and classifications, exclusions, conflicts, permissions/coverage gaps, and questions for an owner.

When the requesting workflow is a team's connector UAT routine (for example, a separate issue-tracking or UAT skill), supply this inventory as input; let that routine decide test coverage and ticket placement. Do not impose a particular tracker's conventions on a team that does not use them. A discovery result is not authorization to change a source or create issues.
