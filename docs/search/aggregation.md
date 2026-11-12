# Search Aggregations

Source (community-verified): https://developer.sailpoint.com/discuss/t/get-users-with-more-than-one-account-in-the-same-source/338

Server-side aggregation over search index data — e.g. counting, grouping, or finding "identities with 2+ accounts in the same source" without pulling every record and filtering client-side. This is a distinct query mode from everything else in `docs/search/` — see [building-query.md](building-query.md) for plain query-string syntax and [audit-events-searching.md](audit-events-searching.md) for DSL-mode filtering; aggregation composes with either as the `query.query` portion, but the response shape and endpoint are different.

## Endpoint

```
POST /search/v1/aggregate
```

**Not** `/search/v1` — that endpoint explicitly rejects an `aggregationsDsl` body with `400: "Aggregations are not allowed in search query."` (confirmed in live testing). `/search/v1/aggregate` is the dedicated endpoint and requires `aggregationsDsl` to be present (`400: "Required field \"aggregationsDsl\" was missing or empty."` if omitted).

## Request shape

```json
{
  "query": { "query": "<query-string, same syntax as building-query.md>" },
  "indices": ["identities"],
  "aggregationsDsl": { "...": "..." }
}
```

- `query.query` scopes which documents feed the aggregation — set to `"*"` for the whole index, or narrow it (recommended, see gotcha below).
- `aggregationsDsl` is raw OpenSearch/Elasticsearch aggregation DSL — `terms`, `nested`, `top_hits`, etc. all work.
- **No pagination.** Aggregation results aren't paged; `limit`/`offset`/`searchAfter` don't apply to the `aggregations` portion of the response (the response also echoes a normal paginated `hits` array for the underlying query, which you can ignore or set `limit: 0` to suppress).

## Worked example: identities with 2+ accounts in the same source

Fields that occur multiple times per record (like an identity's `accounts`) live under a `nested` path in the underlying index — you can't `terms`-aggregate on them directly without first entering the nested context:

```json
{
  "query": { "query": "@accounts(source.name:\"Salesforce SaaS\")" },
  "indices": ["identities"],
  "aggregationsDsl": {
    "accounts": {
      "nested": { "path": "accounts" },
      "aggs": {
        "source_name": {
          "terms": { "field": "accounts.source.name.exact", "min_doc_count": 2, "size": 10 },
          "aggs": {
            "identities": {
              "terms": { "field": "_id", "min_doc_count": 2, "size": 1000 },
              "aggs": { "accounts": { "top_hits": {} } }
            }
          }
        }
      }
    }
  }
}
```

- `min_doc_count: 2` at the `source_name` level keeps only sources where at least 2 nested account docs exist tenant-wide.
- `min_doc_count: 2` at the `identities` level (bucketed by `_id`) is what actually finds *duplicates per identity* — an identity only shows up if 2+ of its nested `accounts` share that source.
- `top_hits` (optional) pulls the actual matching account subdocuments per identity, so you don't need a follow-up call per identity.
- An earlier version of this query (`@accounts(source.name:X) AND accountCount:>2`) is **not equivalent** — `accountCount` counts accounts across *all* sources on the identity, so it false-positives on any identity with one account in the target source plus 2+ elsewhere. The nested-terms approach above only counts accounts within the matched source.

For an authoritative source, scope the outer `query.query` to `source.id:<id>` instead of `@accounts(source.id:<id>)` (authoritative-source accounts are addressed via the identity's own `source` field, not the nested `accounts` collection).

## Gotchas (verified in live testing)

- **Broad query + large nested `size` can return 500.** A large number of nested account documents combined with an unscoped query, `top_hits`, and a large inner `terms` size can exceed a downstream resource or timeout limit. If an aggregation returns 500, narrow the outer query to the relevant source and reduce aggregation `size` values before assuming the feature is unavailable.
- **Same staleness risk as any query against the `accounts` nested field on the `identities` index.** See [search-guide.md](search-guide.md#known-limitations) — the nested `accounts` data can lag or retain orphaned entries relative to live `/accounts/v1` data. An aggregation is not more authoritative than a plain query here; it's just faster to compute. Treat aggregation output as a candidate list and spot-check hits against `/accounts/v1` before reporting counts as current-state fact, especially for anything cleanup/dedup-related where recency matters.
