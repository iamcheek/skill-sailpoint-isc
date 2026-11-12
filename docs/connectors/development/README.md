# Connector development and troubleshooting

Use for an existing or new SailPoint ISC connector build, modification, review or engineering investigation. This is an engineering workflow and source map—not a claim that every connector family, operation or API is documented here. Load the [connector index](../README.md), a matching integration profile, and applicable connector-type/vendor documentation. A team's UAT methodology and issue-tracker structure, when relevant, belong to that team's own process or skills rather than this general engineering guide.

## SaaS development references

| Task | Load |
|---|---|
| Start or continue a connector/customizer interview, research, decision map or implementation spec | [Discovery → specification](discovery.md) and [spec template](spec-template.md). Jira is optional; offline work does not require a tenant. |
| Understand common SDK types, version differences and compound-key example | [Shared SDK foundations](sdk.md). |
| Design or construct a custom SaaS connector | [Connector construction](saas-connectors.md) and [command contracts](commands.md). |
| Design or construct a customizer for a supported connector | [Customizer reference](customizers.md), distinct from connector construction. |
| Test, package or safely validate | [Runtime and testing](runtime-testing.md). |
| Implement an approved slice or review it against a spec | [Implementation and review](implementation-review.md). |

## 1. Discover requirements and architecture

- Establish the business outcome, owners, target systems/environments, in-scope populations, access objects, exclusions and approvals. Gather current requirements and design decisions; do not infer them from code alone.
- For an existing integration, use [configuration discovery](../configuration-discovery.md) and its evidence classifications. For a new integration without a source, inspect approved requirements, target API/vendor documentation and any local prototype instead. Offline work does not need ISC authentication.
- Identify connector family and exact supported documentation/version before choosing a design. A custom SaaS connector, Web Services SaaS source, built-in connector and SCIM target can have materially different capabilities and configuration surfaces.
- Define account and entitlement identities, mappings and correlation; lifecycle and supported operations; permissions/authentication; pagination; rate limits and eventual consistency; failure/retry/partial-write behavior; and audit and security requirements. Record unknowns and alternatives in a profile when requested; don't silently invent behavior.

## 2. Implement and review

- Inspect the actual project's instructions, connector specification, code, tests, dependency versions and deployment/configuration before proposing or changing code. Treat repository state and live source configuration as different evidence; don't assume a local commit is deployed.
- Implement only the approved capability and access scope. Guard identifiers, required attributes and ambiguous records; design retries/idempotency according to the target API's actual contract. Avoid false success when an operation's outcome or a partial write is unknown.
- For a Web Services SaaS integration, consult current vendor documentation and the [generic Web Services SaaS reference](../generic/web-services-saas.md). Verify response roots, schema associations and parent/child operations against the source; observations from one integration are not a universal connector contract.
- An existing integration profile illustrates one implementation. Its keys, joins and lifecycle choices are specific to that integration—not a default design for other connectors.
- Before adding detailed SDK commands, API paths or type-specific instructions, verify against current SailPoint/vendor documentation and the applicable installed SDK. Do not invent a command or claim comprehensive family support from this guide alone. Ask the user when a contract or expected result is unresolved.

## 3. Validate and operate safely

- Run relevant local/automated tests, type checks and packaging checks supported by the project; report exactly what was and wasn't run. Distinguish mocked tests, connector-level validation and live end-to-end acceptance. For live validation establish authorized environment, controlled users, least privilege, safe recovery/cleanup and evidence handling first.
- Local tests and repository changes do not imply permission to change a source, create users, grant access, deploy, rotate credentials, reset data or run disruptive tests. Obtain explicit approval for consequential live operations; follow the parent skill's environment selection and reset/task-status overrides.
- Protect credentials and PII; inspect safe projections rather than dumping source configuration, token-bearing files or raw debug logs. Record version, source and timestamps for observations.
- For source aggregation, account read or provisioning investigations, use the existing [investigation playbooks](../../investigations/README.md) and the relevant connector reference. Test success, persisted objects and account associations are separate facts.
- If project work uses separate issue-tracking or source-control/deployment skills, follow those skills for the team's ticket structure and GitHub/deployment workflows. This guide does not replace them.

## Documentation gaps

The linked SaaS SDK references are a versioned engineering starting point, not a complete or tenant-specific contract. Verify family-specific behavior against the applicable installed SDK and vendor documentation; identify missing documentation or unsupported operations explicitly instead of filling gaps with guessed endpoints.
