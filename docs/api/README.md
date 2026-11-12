# ISC REST API Reference

The shape of the Identity Security Cloud (ISC) REST API — versioning, HTTP methods, endpoints, and error handling. This applies whether you're calling it via `sail api`, an `iscUtils` PowerShell cmdlet, or raw HTTP; for `sail`-specific flag mechanics, see [docs/cli/README.md](../cli/README.md). The optional PowerShell path and its wrapper/version caveats are in [docs/powershell/README.md](../powershell/README.md).

## File map

| File | Use it for |
|---|---|
| [endpoints.md](endpoints.md) | The key endpoints table (including historical identity events), endpoint-specific context, basic GET examples, and the `/search/v1` POST request envelope. |
| [filtering-sorting-pagination.md](filtering-sorting-pagination.md) | The `--query "filters=..."` operator syntax, the verified quoted-date-value bug, sorting, and pagination (including `searchAfter` for >10k results). |
| [response-fields.md](response-fields.md) | Key response fields and enum values per resource (AccountActivity, Identity, Account, AccessProfile, Role). |
| [../roles/](../roles/README.md) | Role-specific endpoint methods, JSON Patch update syntax, membership criteria examples, permissions, and verification. |

Related but separate concerns:
- Query-string syntax for the `events`/`identities`/etc. search indices (Lucene-style `field:term`, nested `@object(...)`, wildcards, `.exact`, `_exists_`) lives in [docs/search/building-query.md](../search/building-query.md) — not duplicated here.
- Transform JSON structure lives in [docs/transform/README.md](../transform/README.md).

## API versioning

ISC has migrated from year-based base-path versioning (`/v2026/...`) to **per-resource path versioning**, e.g. `/accounts/v1`, `/account-activities/v1`, `/access-requests/v1`. Each resource bumps its own version independently. **Use the per-resource form by default** — it's the current source of truth. The old `/v2026/...` paths may still work for some resources but are frozen and no longer receiving updates.

## Method selection

| Intent | Command |
|---|---|
| Fetch/read data | `sail api get` |
| Create a resource | `sail api post` |
| Full update | `sail api put` |
| Partial update | `sail api patch` |
| Delete | `sail api delete` |

## Common errors

| Error | Likely cause | Fix |
|---|---|---|
| 400 Bad Request | Invalid filter operator or syntax | Check the operator is supported for that field — see [filtering-sorting-pagination.md](filtering-sorting-pagination.md). |
| 401 / auth error | Session expired | `sail set token` or check environment config. |
| 403 Forbidden | Missing ISC role/capability | User needs the required ISC permission for that endpoint. |
| 404 Not Found | Wrong path or ID | Confirm the endpoint path and ID are correct — check [endpoints.md](endpoints.md). |

CLI-level errors (command not found, wrong flag for the verb) are covered in [docs/cli/README.md](../cli/README.md#cli-level-errors), not here.
