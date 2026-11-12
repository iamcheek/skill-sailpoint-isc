# SailPoint ISC Search

Search is Identity Security Cloud's (ISC) query interface over an OpenSearch-backed (currently **OpenSearch 2.17**) index of identities, access items, and platform activity. It powers the Search UI, the `sail search query` CLI command, and the `/search/v1` REST API (see the [Key Endpoints table](../../SKILL.md) in the parent skill).

This directory is a reference set for constructing and interpreting search queries: this file is the conceptual map, and each linked file goes deep on one facet.

- Source docs:
  - https://documentation.sailpoint.com/saas/help/search/index.html
  - https://documentation.sailpoint.com/saas/help/search/searchable-fields.html
  - https://documentation.sailpoint.com/saas/help/search/building-query.html
  - https://documentation.sailpoint.com/saas/help/search/search_guide.html
  - https://developer.sailpoint.com/docs/guides/searching-audit-events
  - https://developer.sailpoint.com/docs/guides/audit-events-reference

## File map

| File | Use it for |
|---|---|
| [searchable-fields.md](searchable-fields.md) | The full field reference for every searchable index: `identities`, `entitlements`, `accessprofiles`, `roles`, `events`, `accountactivities` — the only six values the `/search/v1` `index` enum supports. First/second/nested-level field names, types, descriptions. |
| [building-query.md](building-query.md) | Query syntax: field:term basics, AND/OR/NOT, wildcards, ranges, comparisons, `.exact`, `_exists_`, regex, nested `@object(...)` queries, grouping with parentheses. |
| [search-guide.md](search-guide.md) | Operational facts: data latency per index, timezone handling, retention windows, the 10,000-result UI cap, and other gotchas/limitations. |
| [audit-events-searching.md](audit-events-searching.md) | How to query the `events` index specifically for audit/compliance use cases via `/search/v1`, including DSL query bodies, pagination, and common patterns (compliance, security monitoring, role-change tracking, workflow monitoring). |
| [audit-events-reference.md](audit-events-reference.md) | The exhaustive list of every audit event name (`action`/`technicalName`), its search field, its `type` category, and description. |
| [aggregation.md](aggregation.md) | Server-side aggregation via `POST /search/v1/aggregate` (a different endpoint from `/search/v1`) — grouping/counting without pulling records client-side, e.g. "identities with 2+ accounts in the same source." |

---

## What is searchable

ISC Search exposes exactly **six** indices via `/search/v1` — this is the full, authoritative list per the `index` enum in the official OpenAPI spec (`api-specs/idn/apis/shared/schemas/index.yaml` and `search/parameters/index.yaml`); there is no seventh value and the enum note explicitly says additional values may be added in the future without notice, so don't assume a UI tab implies a matching queryable index name:

| Index (API name) | UI tab | Contains |
|---|---|---|
| `identities` | Identities | Correlated identities and their attributes, accounts, apps, access |
| `entitlements` | Entitlements | Entitlements loaded from sources |
| `accessprofiles` | Access Profiles | Access profile configurations |
| `roles` | Roles | Role definitions, assignments, dimensions |
| `events` | Events | Normalized audit events (AUTH, PROVISIONING, CERTIFICATION, etc.) — see [audit-events-searching.md](audit-events-searching.md) |
| `accountactivities` | Account Activity | Provisioning actions on source accounts: access requests, attribute updates, state changes, lifecycle transitions, password changes |

Full field-by-field breakdown of each of the six indices is in [searchable-fields.md](searchable-fields.md).

## Core query concepts

- **Field levels.** A query targets a field with `field:term`. Fields nest: `attributes.location:austin` — `attributes` is a *second-level field* (an object), `location` is the *first-level field* inside it holding the actual value.
- **Active people.** To find currently active people/identities, use `attributes.identityState:ACTIVE`. Do not substitute top-level `status:ACTIVE`; ISC identity-record status is not the person's current active state.
- **No spaces around the colon.** Never write `field: term` or `field :term` — only `field:term`.
- **Nested objects (one-to-many).** Fields that can have multiple instances per record (e.g. an identity's `accounts`, an identity's `access`) are queried with a special `@object(...)` syntax: `@accounts(source.name:Acme)`. See [building-query.md](building-query.md) for the full nested-query rules.
- **Boolean logic.** `AND`, `OR`, `NOT` (all caps) or their symbolic equivalents `&&`, `||`, `!`. Parentheses control evaluation order.
- **Wildcards, ranges, comparisons, exact/regex matching, existence checks.** All covered with literal examples in [building-query.md](building-query.md).

## Where search fits in the platform

- **UI location:** results are viewed under **Search** in the ISC admin console, with a Column Chooser to customize visible fields and a filter icon for category-based filtering.
- **Tagging:** up to 30 tags per object (Identities, Roles, Access Profiles, Entitlements), 500 unique tags site-wide, 10,000 tag associations max. Tags must be 3–128 characters, `A-Z`, `0-9`, and underscores only.
- **AI-assisted query building:** two optional, separately-licensed/enabled paths — Natural Language Search (experimental, AWS Bedrock-backed, opt-in under **Admin > Global > System Settings > Feature Settings > GenAI**, unavailable in FedRAMP Gov Cloud / São Paulo / UAE regions) and Harbor Pilot (licensed feature, requires prompt verification).
- **Downstream uses:** saved searches, certification campaign creation, Separation of Duties (SOD) policy management, audit report download.
- **API/CLI access:** `sail search query "<QUERY>" --indices <INDEX>` (writes to `search_results/`), or `sail api post /search/v1 --body '{...}'` for programmatic access with pagination, sorting, and DSL queries. See the parent [SKILL.md](../../SKILL.md) for CLI usage patterns.

## Operational gotchas (see search-guide.md for full detail)

- Identity/access changes surface in Search almost immediately; other changes (e.g. source edits) can take up to 24 hours via a nightly sync job.
- All date-based search terms must be in **GMT**; the UI displays results in the browser's local timezone regardless.
- UI results cap at **10,000** — use downloadable reports or `searchAfter` pagination via the API to get more.
- Retention: account activity 60 days/2M actions (prod), 30 days/1M actions (staging); audit events 1 year + current month; pending requests indefinite.

## Quick picks by task

- **"What happened" (audit trail, compliance, security events):** [audit-events-searching.md](audit-events-searching.md) + [audit-events-reference.md](audit-events-reference.md)
- **"What field do I filter/sort on for X":** [searchable-fields.md](searchable-fields.md)
- **"How do I write this specific query":** [building-query.md](building-query.md)
- **"Why isn't my query returning what I expect" (timing, limits, timezones):** [search-guide.md](search-guide.md)
