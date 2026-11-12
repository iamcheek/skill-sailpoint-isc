# Generic connector references

Guidance for SailPoint's **generic** connectors: connectors whose vendor guide describes a protocol or file format rather than one application, and which many unrelated integrations share. Examples are Web Services, SCIM 1.1/2.0, JDBC, Delimited File and LDAP. App-specific connectors (Salesforce, Okta, Workday, ServiceNow and so on) have no file here; use their SailPoint guide through [environment connector discovery](../environment-discovery.md) and the integration's [profile](../profile-building.md).

These files sit between the vendor guide and the profile:

| Layer | Answers | Where |
|---|---|---|
| Vendor guide | What the connector documents | SailPoint connector `llms.txt`, fetched on demand |
| Generic reference | How to use the connector family correctly across applications: configuration patterns that span pages, pitfalls, contradictions between vendor pages, connector-level known issues | This folder |
| Integration profile | What one organization's application and sources do | The profile root (see [Profile location](../profile-building.md#profile-location)) |

## Index

| Connector family | File | Covers |
|---|---|---|
| Web Services SaaS | [web-services-saas.md](web-services-saas.md) | Vendor capability boundaries, request construction and retries, group schema wiring, response roots, parent/child operations, pagination and partitioning, endpoint customizers, aggregation verification. |

## Rules for these files

1. **No organization or application data.** No source names or IDs, tenant names, application endpoints or customer-specific observations. Application behavior belongs in the integration's profile. This keeps the files publishable without editing.
2. **One file per connector family.** Label SaaS-based and VA-based content separately, and split into two files only when the content really differs. Name the file after the variant it covers (for example `web-services-saas.md`).
3. **Link the vendor guide; do not copy it.** Record a reviewed date with the vendor guide link. Add guidance only where it adds something the vendor guide does not: cross-page configuration patterns, safety notes, contradictions between vendor pages, and how symptoms appear in ISC.
4. **Promotion from profiles.** A known-issue row tagged **connector-general** in a profile that uses a generic connector can be proposed for the matching file here once it is confirmed (vendor documentation, connector source or a second unrelated integration). Rewrite it without application details, get the user's approval, and link the profile row to the new entry. Do not promote an **unverified** row.
5. **Create files when there is something to say.** Do not add empty stubs. Start a new family's file when the first confirmed, connector-general guidance exists, and add it to the index above.
