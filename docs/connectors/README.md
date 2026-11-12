# Connector engineering references

Use this index for connector design/builds, configuration discovery, integration profiles and troubleshooting. Load the relevant file on demand. Vendor connector documentation, generic connector guidance, an application-specific profile and live source configuration are different things.

## Engineering workflows

| Task | Reference |
|---|---|
| Inventory every source in a tenant, match each to its SailPoint connector guide, and draft or refresh integration profiles | [Environment connector discovery](environment-discovery.md) — read-only ISC access; writes local profile files only after approval. |
| Inventory an existing source's schemas, policies, mappings and capabilities for planning or troubleshooting | [Configuration discovery](configuration-discovery.md) — read-only; reports unknowns and provenance. |
| Create, review or update a durable integration profile | [Profile-building routine](profile-building.md) — synthesize requirements, rationale, implementation and dated observations. |
| Start or resume a SaaS connector/customizer interview, discovery, decision map or repository spec | [Discovery → specification](development/discovery.md) — offline-capable technical workflow; use your team's issue-tracking process for any requested tracker operations. |
| Design, implement, test or troubleshoot a connector or customizer | [Development guide](development/README.md) — SDK, connector, customizer, command, testing and handoff references. |

## Integration profiles

No integration profiles are bundled with this skill. Profiles describe one organization's implementation (its sources, configuration, requirements, design decisions and known issues), so they live under a profile root you choose, with one folder per organization (see [Profile location](profile-building.md#profile-location)). Use [environment connector discovery](environment-discovery.md) to draft profiles for every source in a tenant, or the [profile-building routine](profile-building.md) for one integration.

Profile location, organization folders and `tenants.md` are defined in [profile-building](profile-building.md#profile-location). Load a matching integration profile whenever its connector is identified and relevant, including planning, development and UAT—not just after a failure. Match aliases, source, environment and version before applying claims; label conflicting or stale facts. If no profile exists, report that gap and use [profile-building](profile-building.md) when asked to create one.

## Generic connector references

[generic/](generic/README.md) holds guidance for connectors that describe a protocol or format rather than one application (Web Services, SCIM, JDBC, Delimited File and similar), which many integrations share. Load the matching file whenever a source uses one of these connectors, alongside the integration profile. App-specific connectors (Salesforce, Okta, Workday and so on) have no generic file; their guidance comes from the SailPoint guide and the profile.

| Connector family | File |
|---|---|
| Web Services SaaS | [generic/web-services-saas.md](generic/web-services-saas.md) |

## Where troubleshooting findings go

A confirmed troubleshooting finding goes into the integration profile's [known issues](profile-building.md#known-issues), tagged connector-general, integration-specific or unverified. Connector-general findings on a generic connector can then be [promoted](generic/README.md#rules-for-these-files) into the generic file. See [../investigations/provisioning.md](../investigations/provisioning.md#connector-specific-troubleshooting) for how investigations use these files.
