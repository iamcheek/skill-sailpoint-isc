# Filtering, Sorting, and Pagination (`sail api get` query params)

This covers the `--query "filters=..."` / `"sorters=..."` operator syntax used with `sail api get` (and other verbs — see [docs/cli/README.md](../cli/README.md) for which verbs accept `--query`). For the Lucene-style query-string language used in `sail search query` and inside a `/search/v1` POST body's `query.query`, see [docs/search/building-query.md](../search/building-query.md) instead — these are two different syntaxes.

## Filter syntax (`--query "filters=..."`)

Format: `field operator "value"` — string values in double quotes, booleans/numbers unquoted.

> **Verified in live testing:** Quoting a date-time value with `gt`/`ge`/`lt`/`le` (e.g. `created ge "<timestamp>"`) silently returns 0 results/count instead of erroring — confirmed on both `/account-activities/v1` and `/access-request-administration/v1`. **Do not quote date-time values.** Drop the quotes (`created ge <timestamp>`) and the filter works correctly. String `eq`/`sw`/`co` filters (e.g. `type eq "appRequest"`) are unaffected and still require quotes. If a date/time filter returns a suspiciously low or zero count, re-check for quotes before concluding the field or the record set is empty.

**Operators:**

| Operator | Meaning | Example |
|---|---|---|
| `eq` | equals | `name eq "John Doe"` |
| `ne` | not equals | `correlated ne false` |
| `sw` | starts with | `alias sw "john"` |
| `co` | contains | `name co "admin"` |
| `gt` | greater than | `created gt 2024-01-01` |
| `ge` | greater than or equal | `created ge 2024-01-01` |
| `lt` | less than | `modified lt 2025-01-01` |
| `le` | less than or equal | `modified le 2025-12-31` |
| `in` | in list | `id in ("abc123","def456")` |
| `isnull` | is null | `completionStatus isnull` |
| `and` | logical AND | `type eq "appRequest" and created gt 2024-01-01` |
| `or` | logical OR | `id eq "abc" or correlated eq false` |
| `not` | logical NOT (with `in` only) | `not value in ("ID01","ID02")` |

**Notes:**
- For `/accounts/*`, use the account resource's flat `sourceId` filter (for example, `sourceId eq "<source-id>"`). Do not assume the entitlement filter path `source.id` works for accounts; `source.id` is the documented form for `/entitlements/*`.
- Only a single `and` or `or` per expression; must join two different fields.
- `not` can only precede `in`: `not value in (...)`.
- Not every operator applies to every field — if you get a 400 error, the operator may not be supported for that field.
- Date/time values passed to `gt`/`ge`/`lt`/`le` must be **unquoted** (see callout above) — this applies even inside an `and`/`or` expression.

**Filter examples by resource:**

```bash
# Account activities — by type
sail api get /account-activities/v1 \
  --query 'filters=type eq "appRequest"' --pretty

# Account activities — type AND date range (note: unquoted date)
sail api get /account-activities/v1 \
  --query 'filters=type eq "Identity Refresh" and created gt 2024-01-01' --pretty

# Access requests submitted in the last 24 hours (org-wide, unquoted date)
sail api get /access-request-administration/v1 -H 'X-SailPoint-Experimental: true' \
  --query 'filters=created ge <timestamp>' --query "count=true" --pretty

# Accounts — by identity ID
sail api get /accounts/v1 \
  --query 'filters=identityId eq "2c9180858082150f0180893dbaf44201"' --pretty

# Accounts — by native identity (account UUID on the target system) — useful when pivoting
# from a connector log or account-activity item straight to the account record
sail api get /accounts/v1 \
  --query 'filters=nativeIdentity eq "11111111-2222-3333-4444-555555555555"' --pretty

# Accounts — by source ID (use sourceId on /accounts/v1)
sail api get /accounts/v1 \
  --query 'filters=sourceId eq "<source-id>"' --pretty

# Accounts — by source display name
sail api get /accounts/v1 \
  --query 'filters=source.displayableName eq "Active Directory"' --pretty

# Note: entitlement filters use source.id, not sourceId
sail api get /entitlements/v1 \
  --query 'filters=source.id eq "<source-id>"' --pretty

# Identities — starts with
sail api get /identities/v1 \
  --query 'filters=name sw "John"' --pretty

# Entitlements — by source
sail api get /entitlements/v1 \
  --query 'filters=source.id eq "2c9180887671ff8c0176883f9ff52081"' --pretty
```

## Sorting (`--query "sorters=..."`)

Comma-separated field names. Prefix `-` for descending, `+` or nothing for ascending.

```bash
sail api get /accounts/v1 --query "sorters=name" --pretty
sail api get /account-activities/v1 --query "sorters=-created" --pretty
sail api get /identities/v1 --query "sorters=name,-modified" --pretty
```

## Pagination

Standard list endpoints: max `limit=250`. Roles: max `limit=50`.

```bash
# Page through results
sail api get /accounts/v1 --query "limit=250" --query "offset=0" --pretty
sail api get /accounts/v1 --query "limit=250" --query "offset=250" --pretty
```

**Pagination past 10,000 records** (via `/search/v1` POST, see [endpoints.md](endpoints.md#search-via-post-searchv1)) — use `searchAfter` with the last record's sorted field values:

```bash
sail api post /search/v1 --pretty --body '{
  "indices": ["identities"],
  "query": { "query": "*" },
  "sort": ["displayName", "+id"],
  "searchAfter": ["John Doe", "2c91808375d8e80a0175e1f88a575221"],
  "limit": 250
}'
```
