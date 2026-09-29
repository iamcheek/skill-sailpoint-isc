# Key Endpoints

See [README.md](README.md) for versioning conventions and method selection. See [filtering-sorting-pagination.md](filtering-sorting-pagination.md) for `--query "filters=..."` syntax, and [response-fields.md](response-fields.md) for what each resource returns.

## Key endpoints table

| Resource | Path | Notes |
|---|---|---|
| Account Activities (list) | `/account-activities/v1` | Filter by `type`, `created`, `modified` |
| Account Activity (single) | `/account-activities/v1/{id}` | |
| Identities (list) | `/identities/v1` | Filter by `name`, `alias`, `email`, `correlated` |
| Accounts (list) | `/accounts/v1` | Filter by `identityId`, `sourceId`, `name`, `hasEntitlements` |
| Entitlements (list) | `/entitlements/v1` | Filter by `source.id`, `name`, `type`, `requestable` |
| Entitlements (list, beta) | `/beta/entitlements` | Beta/UI-backed entitlement data path; behavior can differ from `/entitlements/v1` |
| Entitlement (single, beta) | `/beta/entitlements/{id}` | Beta/UI-backed detail lookup |
| Access Profiles (list) | `/access-profiles/v1` | Filter by `name`, `source.id`, `requestable` |
| Roles (list) | `/roles/v1` | Filter by `name`, `requestable`; max 50 per page |
| Sources (list) | `/sources/v1` | |
| Search | `/search/v1` | POST with JSON body (see below); also handles `accountactivities` index |
| Access Requests (submit) | `/access-requests/v1` | POST only, body required. `requestType`: `GRANT_ACCESS` or `REVOKE_ACCESS`. `requestedFor[]` (identity IDs) + `requestedItems[]`, or `requestedForWithRequestedItems[]` for multi-account/per-item start/remove dates (GRANT only). Async — returns 202, not an error. |
| Access Requests (cancel) | `/access-requests/v1/cancel` | POST. Body: `accountActivityId`, `comment`. Only pre-approval; requester or ORG_ADMIN. |
| Access Request Status (list, self-scoped) | `/access-request-status/v1` | GET. Query params `requested-for`, `requested-by`, `regarding-identity`, `assigned-to` (`me` = current user); `request-state=EXECUTING`. `filters=` supports `accessRequestId`, `accountActivityItemId`, `created`. Any user can see their own; ORG_ADMIN needed for others. |
| Access Request Status (list, admin/org-wide) | `/access-request-administration/v1` | GET, ORG_ADMIN or `idn:access-request-administration:read`. **Requires header** `X-SailPoint-Experimental: true`. Same params as above, plus `filters=status`. |
| Identity Processing (bulk reprocess) | `/identities/v1/process` | POST. Body: `{"identityIds": [...]}`, 1–250 per call. **Requires header** `X-SailPoint-Experimental: true`. ORG_ADMIN/HELPDESK. Recalculates attributes, evaluates role assignments, and **enforces provisioning for previously-failed items** — the right tool for manually retrying a failed grant after a transient source-side error clears. Returns 202 + `{type: TASK_RESULT, id}` — see the task-tracking gotcha below before trying to poll that `id`. Full remediation workflow (identityId resolution, verification via `events`, batch caveats) in [docs/investigations/provisioning.md](../investigations/provisioning.md#remediation-manually-reprocessing-an-identity-after-a-resolvedtransient-failure). |
| Identity Processing (single-identity attribute sync) | `/identities/v1/{id}/synchronize-attributes` | POST. **Requires header** `X-SailPoint-Experimental: true`. ORG_ADMIN. Rate-limited to once per 10s per identity. Attribute recalculation only — does **not** retry failed role/entitlement provisioning, so it's the wrong endpoint for the provisioning-retry use case above; use `/identities/v1/process` instead. |
| Task Status (single, by ID) | `/task-status/v1/{id}` | GET. Covers classic background-task types (account/entitlement aggregation, etc.) — IDs are 32-char hex, no dashes. **Does not track `/identities/v1/process` jobs** — that endpoint's returned `id` is a dashed UUID in a different ID namespace and 404s here. See the provisioning doc link above for event-based verification instead. |
| Task Status (list) | `/task-status/v1` | GET. `filters=` supports `id` (`eq`/`in`), `sourceId`, `completionStatus` (`eq`/`in`/`isnull`), `type`, `launcher`. Data older than 90 days not returned. |

## Beta entitlement endpoints

The beta entitlement paths may expose data through a newer or UI-backed path than
`/entitlements/v1`. They are not interchangeable version aliases. When a newly completed
aggregation is not visible through the normal endpoint, check the beta path only if the tenant,
permissions, and installed API behavior support it. Compare the source ID, aggregation status,
entitlement association, and response from both paths before concluding that the object was not
persisted. Do not replace all v1 calls with beta calls solely because a beta path is available.

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
