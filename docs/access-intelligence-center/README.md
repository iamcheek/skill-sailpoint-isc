# Access Intelligence Center (AIC)

Access Intelligence Center is ISC's BI/reporting layer, built on embedded **Qlik Sense** (Qlik Cloud). It surfaces identity, access, request, and certification data as interactive dashboards and lets admins build their own visualizations ("sheets") against that data.

- Source docs: https://documentation.sailpoint.com/saas/help/ai/access_insights/access_intelligence.html
- Data model reference: [er-diagram.md](er-diagram.md) — the entities/fields/relationships behind every dashboard and custom sheet.

**Out of scope for `sail`/`sail api`.** Unlike search, transforms, workflows, and rules, AIC has no REST API or CLI surface documented here — it's managed entirely through the Qlik-based web UI (**Home > Access Intelligence Center**). This file is knowledge for reasoning about the feature and its data model, not a command reference.

---

## Dashboards

AIC ships five pre-built dashboards, each backed by a different slice of the data model:

| Dashboard | Focus |
|---|---|
| **Access Intelligence Center** | Identity program overview — identities, requests, certifications, and access relationships. |
| **AIC Audit** | Tangible audit events — access requests, certifications, lifecycle state changes, entitlement add/remove, account create/delete. Last 3 months or 5M events, whichever is smaller. |
| **Non-Employee Risk Management** | Non-employee and assignment data. Requires the relevant person/assignment-profile account attributes to be [synced to AIC](https://documentation.sailpoint.com/ne-admin/help/connector/index.html#synchronizing-data-to-the-access-intelligence-center) first. |
| **Data Access Security** | On-boarded applications in Data Access Security — account info, application resources, permissions, and data classification. |
| **SailPoint CIEM** | Cloud infrastructure entitlement details — cloud resources/services, actions, usage data. Joins with entitlement data for least-privilege sizing. |

Applications not licensed in the tenant show disabled — that's expected, not a bug; contact the CSM to license them.

## Sheets, visualizations, and permissions

- A **sheet** is a page of **visualizations** (charts/extensions) built from **measures** and **dimensions** — see [Qlik's visualization docs](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/Visualizations/visualizations-in-app.htm) for the underlying concepts.
- Sheets can be built three ways: exploring data directly, asking the Insight Advisor a natural-language question, or hand-building visualizations with explicit dimensions/measures.
- **Reader** permission: view public sheets, filter them. **Author** permission: additionally create public/private sheets and bookmark filters. Custom User Levels can grant AIC Reader/Author individually; standard user levels require Admin or Report Admin.
- Private sheets live under **My sheets**; publishing makes a sheet public and visible to everyone.
- 5GB total size cap per tenant (raw data + sheets + bookmarks + users combined).

## Operational gotchas

- **Latency**: ISC data changes can take **up to 24 hours** to reflect in AIC dashboards — don't expect real-time parity with Search or the API.
- **Browser support**: third-party cookies must be enabled; **Safari is not supported**.
- **Firewall allowlisting**: strict-allowlist customers must permit `cdn.qlikcloud.com:443`.
- **Canada-hosted tenants**: use the Qlik Sense (not Qlik Cloud) documentation links throughout — the UI is a different Qlik product version.
- Public sheets/charts can be surfaced as widgets on the MySailPoint home dashboard (Atlas Enterprise license required) via **Monitor in activity center** — this uses a scoped OAuth client credential (`QLIK-BIR`, `sp:monitored-charts:refresh`) that only triggers a refresh, no customer data passed.

## When to use this vs. Search/API

AIC is for **cross-entity, aggregate, trend, and audit reporting** over history — the kind of "how many," "over time," "distribution across" question a dashboard answers better than a query. For a single-identity or single-request lookup, or anything needing live/current state, use [docs/search/](../search/README.md) or [docs/api/](../api/README.md) instead — they're faster and not subject to AIC's up-to-24h sync lag.
