# Search Guide — Operational Facts & Limitations

Source: https://documentation.sailpoint.com/saas/help/search/search_guide.html

Operational realities that explain "why didn't my query return what I expected" — timing, timezones, retention, and hard limits. For syntax, see [building-query.md](building-query.md); for fields, see [searchable-fields.md](searchable-fields.md).

## Data latency by index

Search is **not a live view of the platform** — it's a periodically-reindexed copy. Per SailPoint's own docs: "Data changes that impact identities and their access are usually accessible in Search immediately or within a few minutes, though in a busy environment, longer delays may occur." Everything else rides a slower, scheduled path.

| Change type | Time to appear in Search |
|---|---|
| Identity and access changes | Immediately, or within a few minutes — longer in a busy environment |
| Other updates (e.g. source edits, entitlement/access-application metadata) | Up to 24 hours — processed via a nightly `SYNCHRONIZE_IDENTITIES` job |

If a query seems to be missing a very recent non-identity/access change, it may simply not have synced yet — recheck after the nightly job rather than assuming the query is wrong.

**Where account-correlation changes fall is ambiguous, and evidence points to more lag than the "few minutes" figure suggests.** A source account gaining/losing its correlation to an identity is arguably an "access change," but the nested `accounts` field on an identity's `identities`-index document is exactly what we found stale in practice (see the known limitation below) — including account IDs that no longer resolve at all via `/accounts/v1`. Treat the latency table above as a floor, not a guarantee, for anything involving an identity's `accounts`/correlation state specifically — verify against the live `/accounts/v1` API before trusting a nested-`accounts` read as current.

## Timezone handling

- All date values used **in a search query** must be expressed in **GMT**.
- Search **results** are displayed in the browser's local timezone.
- This means: if you construct a query with a date range and the numbers look "off" by your local UTC offset, you likely need to convert your intended local time to GMT before writing the query — the display conversion only happens after the fact, not for query input.

## Data retention

| Data type | Retention |
|---|---|
| Account activity (production) | 60 days, or 2,000,000 actions — whichever comes first |
| Account activity (staging) | 30 days, or 1,000,000 actions — whichever comes first |
| Audit events | 1 year plus the current month |
| Pending requests | Indefinite |

Queries against account activity or events older than these windows will not return results even if the underlying action genuinely occurred — the data has aged out, not failed to index.

## Known limitations

- **UI display cap: 10,000 results.** The Search UI will not display more than 10,000 matching records no matter how broad the query. Use a downloadable report to bypass this, or use `searchAfter` pagination via the `/search/v1` API (see the parent skill's [SKILL.md](../../SKILL.md) for the `searchAfter` pattern).
- **A high `limit` on `/search/v1` does not guarantee you got everything up to it.** Verified in live testing: a `/search/v1` POST with `limit: 10000` explicitly set returned exactly 250 results — but `?count=true` on the endpoint, read via the `X-Total-Count` response header, showed a much larger total. Don't infer "that's everything" from an error-free response with a plausible-looking count — always check `X-Total-Count` when the count matters, e.g. `sail --debug api post "/search/v1?count=true" --body '...' 2>&1 | grep -i x-total-count`.
- **Time-windowing a search can silently truncate an incident's real duration.** Verified in live testing: a search windowed to a plausible-looking time range showed the issue stopping partway through; removing the window showed it continued substantially longer. A clean, error-free result inside an arbitrary window is not evidence the window was wide enough — widen or remove the window before concluding an issue's timeline from a bounded query.
- **Certain symbols aren't searchable directly.** `#` and `+` cannot be matched literally in a plain query string — use `?` (single-character wildcard) as a substitute, or `.exact` with a regex (see [building-query.md](building-query.md)).
- **`_exists_` only works reliably against fields matching their documented format.** If a field's actual stored value doesn't conform to the type/format listed in [searchable-fields.md](searchable-fields.md), an `_exists_` check against it may behave unexpectedly.
- **`manager.name` searches are bidirectional.** See the gotcha noted in [building-query.md](building-query.md) — querying for a manager's name also returns that manager's own identity record, not just their reports.
- **The nested `accounts` field on the `identities` index can be stale or retain orphaned entries.** Verified in live testing: an identity's nested `accounts` list included stale account IDs that returned 404 on direct `/accounts/v1/{id}` lookup, while `/accounts/v1` showed the current account set. This is consistent with reindexing lag rather than a distinct bug. Don't trust nested-`accounts` counts (from a plain query or an aggregation, see [aggregation.md](aggregation.md)) as current-state fact for anything correlation/dedup-sensitive — treat them as a candidate list and confirm against `/accounts/v1` before reporting.

## Technical foundation

- The search engine runs on **OpenSearch 2.17**.
- For anything the simplified query-string syntax in [building-query.md](building-query.md) can't express, the underlying engine's **Query DSL**, **Query String Syntax**, and **Regular Expression Syntax** are available — see [audit-events-searching.md](audit-events-searching.md) for a worked example of switching to DSL mode (`queryType: "DSL"`) via the `/search/v1` API.
- For server-side grouping/counting (e.g. "identities with 2+ accounts in the same source") without paging through records client-side, a separate aggregation endpoint is available — see [aggregation.md](aggregation.md).

## Sample data models

The source page includes full JSON sample documents for each searchable type: identity (with accounts/apps/access), entitlement (with privilege levels and metadata), access profile, role (with dimensions/segments/access model metadata), event (audit trail), account activity (provisioning requests/approvals/expansions). For the field-level breakdown of each (rather than a full sample document), see [searchable-fields.md](searchable-fields.md).
