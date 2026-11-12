# Key Endpoints

See [README.md](README.md) for versioning conventions and method selection. See [filtering-sorting-pagination.md](filtering-sorting-pagination.md) for `--query "filters=..."` syntax, and [response-fields.md](response-fields.md) for what each resource returns.

## Key endpoints table

| Resource | Path | Notes |
|---|---|---|
| Account Activities (list) | `/account-activities/v1` | Filter by `type`, `created`, `modified` |
| Account Activity (single) | `/account-activities/v1/{id}` | |
| Identities (list) | `/identities/v1` | Filter by `name`, `alias`, `email`, `correlated` |
| Historical identity events (list) | `GET /historical-identities/v1/{id}/events` | Identity-specific access history; requires `X-SailPoint-Experimental: true` and identity-history authorization. Optional `from`, `eventTypes`, `accessItemTypes`, `limit`, `offset`, `count`; see below. |
| Accounts (list) | `/accounts/v1` | Filter by `identityId`, `sourceId`, `name`, `hasEntitlements` |
| Reload one account (one-time aggregation) | `POST /accounts/v1/{id}/reload` | Asynchronously reloads one account directly from its connector and runs a one-time aggregation; `202` returns a task ID. Requires scope `idn:accounts-state:manage`; user levels `ORG_ADMIN`, `SOURCE_ADMIN`, `SOURCE_SUBADMIN`, or `HELPDESK`. See [source-aggregation.md](../investigations/source-aggregation.md#single-account-reload). |
| Entitlements (list) | `/entitlements/v1` | Filter by `source.id`, `name`, `type`, `requestable` |
| Entitlements (list, beta) | `/beta/entitlements` | Beta/UI-backed entitlement data path; behavior can differ from `/entitlements/v1` |
| Entitlement (single, beta) | `/beta/entitlements/{id}` | Beta/UI-backed detail lookup |
| Access Profiles (list) | `/access-profiles/v1` | Filter by `name`, `source.id`, `requestable` |
| Roles (list) | `/roles/v1` | Filter by `name`, `requestable`; max 50 per page |
| Sources (list) | `/sources/v1` | |
| Account aggregation | `POST /sources/v1/{id}/load-accounts` | Multipart form; optional `disableOptimization=true` reprocesses every account. See [source-aggregation.md](../investigations/source-aggregation.md). |
| Entitlement aggregation | `POST /sources/v1/{sourceId}/load-entitlements` | Multipart form; a CSV file may be needed for Delimited File sources. See [source-aggregation.md](../investigations/source-aggregation.md). |
| Source accounts reset (all accounts) | `POST /sources/v1/{id}/remove-accounts` | Destructive and asynchronous; removes accounts from ISC without provisioning changes to the source. Typically sandbox/new-source work; explicit user confirmation for the exact environment/source is mandatory. |
| Single account removal from ISC | `POST /accounts/v1/{id}/remove` | Asynchronous; removes one account from ISC without provisioning changes to the source. It may return on a later aggregation. A returned task ID may not be trackable through `/task-status/v1/{id}`; verify the account directly. See [source-reset.md](../investigations/source-reset.md) for the tightly scoped source-owner-blocker case. |
| Source entitlements reset | `POST /entitlements/v1/reset/sources/{id}` | Destructive and asynchronous; removes all entitlements. Typically sandbox/new-source work; explicit user confirmation for the exact environment/source is mandatory. See [source-reset.md](../investigations/source-reset.md) for safe sequencing and reaggregation guidance. |
| Search | `/search/v1` | POST with JSON body (see below); also handles `accountactivities` index |
| Access Requests (submit) | `/access-requests/v1` | POST only, body required. `requestType`: `GRANT_ACCESS` or `REVOKE_ACCESS`. `requestedFor[]` (identity IDs) + `requestedItems[]`, or `requestedForWithRequestedItems[]` for multi-account/per-item start/remove dates (GRANT only). Async — returns 202, not an error. |
| Access Requests (cancel) | `/access-requests/v1/cancel` | POST. Body: `accountActivityId`, `comment`. Only pre-approval; requester or ORG_ADMIN. |
| Access Request Status (list, self-scoped) | `/access-request-status/v1` | GET. Query params `requested-for`, `requested-by`, `regarding-identity`, `assigned-to` (`me` = current user); `request-state=EXECUTING`. `filters=` supports `accessRequestId`, `accountActivityItemId`, `created`. Any user can see their own; ORG_ADMIN needed for others. |
| Access Request Status (list, admin/org-wide) | `/access-request-administration/v1` | GET, ORG_ADMIN or `idn:access-request-administration:read`. **Requires header** `X-SailPoint-Experimental: true`. Same params as above, plus `filters=status`. |
| Identity Processing (bulk reprocess) | `/identities/v1/process` | POST. Body: `{"identityIds": [...]}`, 1–250 per call. **Requires header** `X-SailPoint-Experimental: true`. ORG_ADMIN/HELPDESK. Recalculates attributes, evaluates role assignments, and **enforces provisioning for previously-failed items** — the right tool for manually retrying a failed grant after a transient source-side error clears. Returns 202 + `{type: TASK_RESULT, id}` — see the task-tracking gotcha below before trying to poll that `id`. Full remediation workflow (identityId resolution, verification via `events`, batch caveats) in [docs/investigations/provisioning.md](../investigations/provisioning.md#remediation-manually-reprocessing-an-identity-after-a-resolvedtransient-failure). |
| Identity Processing (single-identity attribute sync) | `/identities/v1/{id}/synchronize-attributes` | POST. **Requires header** `X-SailPoint-Experimental: true`. ORG_ADMIN. Rate-limited to once per 10s per identity. Attribute recalculation only — does **not** retry failed role/entitlement provisioning, so it's the wrong endpoint for the provisioning-retry use case above; use `/identities/v1/process` instead. |
| Task Status (single, by ID) | `/task-status/v1/{id}` | GET. Covers classic background-task types (account/entitlement aggregation, etc.) — IDs are 32-char hex, no dashes. **Does not track `/identities/v1/process` jobs** — that endpoint's returned `id` is a dashed UUID in a different ID namespace and 404s here. See the provisioning doc link above for event-based verification instead. |
| Task Status (update, by ID) | `PATCH /task-status/v1/{id}` | JSON Patch. Can change `completed` and `completionStatus`, including clearing a pending-task blocker. High-risk manual write; see [stale-task-remediation.md](../investigations/stale-task-remediation.md), verify the task outcome, and obtain explicit confirmation before updating. |
| Task Status (list) | `/task-status/v1` | GET. `filters=` supports `id` (`eq`/`in`), `sourceId`, `completionStatus` (`eq`/`in`/`isnull`), `type`, `launcher`. Data older than 90 days not returned. |

## Historical identity events

`GET /historical-identities/v1/{id}/events` returns a paginated array of historical events for one identity. This is the Identity History API operation [`getHistoricalIdentityEventsV1`](https://developer.sailpoint.com/docs/api/get-historical-identity-events-v-1), not a query against the `/search/v1` `events` index.

The endpoint is experimental; include the required header and an identity-history authorization scope:

```bash
sail --env <name> api get /historical-identities/v1/<identity-id>/events \
  -H 'X-SailPoint-Experimental: true' --pretty
```

The OpenAPI description names `idn:identity-history:read` as the required scope; its security declaration lists the `idn:identity-history:read` and `idn:identity-history:manage` scopes for user and application authentication.

Optional query parameters:

- `from` — timestamp parameter; the spec describes it as the optional instant until which access events are returned. The spec does not define a `to` parameter.
- `eventTypes` — optional array of event-type strings; omitted or empty returns all events. The spec's examples are `AccessAddedEvent` and `AccessRemovedEvent`.
- `accessItemTypes` — optional array (examples: `entitlement`, `account`); omitted or empty returns all access-item types.
- `limit`, `offset`, `count` — standard list pagination/count parameters.

The response is a polymorphic event array. Examples in the spec include `IdentityCertified`, `AccessItemAssociated`, `AccessItemRemoved`, `AttributesChanged`, `AccessRequested`, and `AccountStatusChanged`; each has common fields such as `eventType`, `identityId`, and `dt`, plus event-specific data. Use this endpoint for history associated with a particular identity; use the `events` search index for DSL-based cross-identity audit or compliance searches.

## Beta entitlement endpoints

The beta entitlement paths are worth checking when a newly completed connector aggregation appears
in the ISC UI or connector logs but the current per-resource entitlement endpoint does not show the
objects. These are not interchangeable version aliases: a beta path can be backed by a newer data
path than `/entitlements/v1`.

**Observed in live testing (not a universal propagation guarantee):** for one newly configured
Web Services source, `/beta/entitlements` returned the aggregated entitlements while
`/entitlements/v1` and `/v2026/entitlements` returned zero, and a known entitlement ID returned 200
from `/beta/entitlements/{id}` but 404 from `/entitlements/v1/{id}`. Therefore, a v1 404 or a zero
result does not, by itself, prove that a newly aggregated entitlement was not persisted. Check the
beta/UI-backed path, source ID, aggregation completion, and the account association separately.

Use the beta paths only when the tenant/API behavior and permissions support them; do not replace
all v1 calls with beta calls solely because a beta path is available.

## Basic GET examples

```bash
# List accounts (pretty printed)
sail api get /accounts/v1 --pretty

# Single account activity by ID
sail api get /account-activities/v1/abc123 --pretty

# With limit and offset
sail api get /accounts/v1 --query "limit=50" --query "offset=0" --pretty

# Include total count in response header
sail api get /identities/v1 --query "count=true" --pretty
```

## Search via POST (`/search/v1`)

Use `sail api post` for complex searches with body filters. The documented max is `limit=10000` (see [filtering-sorting-pagination.md](filtering-sorting-pagination.md#pagination) for paging past that with `searchAfter`), but don't trust `limit` alone to mean you got everything up to it — confirmed in live testing: a query with `limit: 10000` explicitly set still returned only 250 results for one identity-scoped query, while `?count=true` + the `X-Total-Count` response header revealed a much larger true total. Always check `X-Total-Count` when a result count matters, rather than trusting the returned array length. See [search-guide.md](../search/search-guide.md#known-limitations) for the general gotcha.

```bash
sail api post /search/v1 --pretty --body '{
  "indices": ["identities"],
  "query": { "query": "attributes.city:London" },
  "filters": {
    "source.name": { "type": "TERMS", "terms": ["HR Employees"] },
    "isManager": { "type": "TERMS", "terms": ["true"] }
  },
  "sort": ["displayName", "+id"],
  "limit": 250,
  "offset": 0
}'
```

The string inside `query.query` uses the same query-string language documented in [docs/search/building-query.md](../search/building-query.md) (field:term, nested `@object(...)`, wildcards, `.exact`, `_exists_`, etc.) — that syntax is not repeated here.

For DSL-mode queries against the `events` index specifically (compliance/security/audit use cases), see [docs/search/audit-events-searching.md](../search/audit-events-searching.md).

For server-side grouping/counting instead of paging through raw records (e.g. "identities with 2+ accounts in the same source"), a separate endpoint — `POST /search/v1/aggregate`, not `/search/v1` — accepts an `aggregationsDsl` body. See [docs/search/aggregation.md](../search/aggregation.md).

**Filter types in the POST body** (distinct from the `--query "filters=..."` operator syntax used with GET — see [filtering-sorting-pagination.md](filtering-sorting-pagination.md)):

```json
// TERMS — exact match list
{ "type": "TERMS", "terms": ["value1", "value2"], "exclude": false }

// RANGE — date or numeric
{ "type": "RANGE", "range": { "lower": { "value": "2023-12-01", "inclusive": true }, "upper": { "value": "2025-01-01" } } }

// EXISTS — field presence check
{ "type": "EXISTS", "exclude": false }
```
