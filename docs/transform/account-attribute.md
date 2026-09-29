# Account Attribute

**Transform type:** `accountAttribute`

Retrieves a specific attribute value from an account associated with a particular source on an identity. When an identity has multiple accounts on the source, the transform defaults to using the oldest account (by creation date), with configurable sorting and filtering options to change which account's value is returned.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `sourceName` | Yes (or use `applicationId`/`applicationName`) | string | The source's display name (e.g., "Active Directory"). Alternatives to specify the source are `applicationId` or `applicationName`. |
| `attributeName` | Yes | string | The account attribute name to retrieve. |
| `accountSortAttribute` | No | string | Schema attribute used to sort accounts when the identity has multiple accounts on the source. Default: `"created"`. |
| `accountSortDescending` | No | boolean | Sort order for accounts; `true` = newest first. Default: `false`. |
| `accountReturnFirstLink` | No | boolean | If `true`, returns the first account's value even if it is null. If `false`, returns the first non-null value found across accounts. Default: `false`. |
| `accountFilter` | No | string | Database query filter expression (`sailpoint.object.Filter` syntax), evaluated at the database level. Searchable by `nativeIdentity`, `displayName`, `entitlements`, etc. |
| `accountPropertyFilter` | No | string | In-memory filter expression evaluated after accounts are retrieved; all account attributes are available for filtering. |
| `input` | No | object | Explicit input data override (e.g., used to nest this transform inside another transform's input). |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate this transform's logic during the nightly identity refresh. Default: `false`. |

**Notes on filters:** `accountFilter` searches the database directly; `accountPropertyFilter` filters in-memory after accounts are retrieved. Both filters are combined with AND logic alongside the default source/identity filters.

## Example(s)

Example 1 — HR system lookup sorted newest-first with an active-worker in-memory filter:
```json
{
  "attributes": {
    "attributeName": "HIREDATE",
    "sourceName": "Corporate HR",
    "accountSortAttribute": "created",
    "accountSortDescending": true,
    "accountReturnFirstLink": true,
    "accountPropertyFilter": "(WORKER_STATUS__c == \"active\")"
  },
  "type": "accountAttribute",
  "name": "Account Attribute Transform"
}
```

Example 2 — Active Directory username lookup excluding service accounts via a database-level filter:
```json
{
  "attributes": {
    "attributeName": "sAMAccountName",
    "sourceName": "Active Directory",
    "accountFilter": "!(displayName.startsWith(\"SVC-\"))"
  },
  "type": "accountAttribute",
  "name": "Account Attribute Transform"
}
```

Example 3 — Nested inside a `lookup`/`firstValid` combination to check entitlement membership and produce a TRUE/FALSE flag:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "values": [
          {
            "attributes": {
              "accountPropertyFilter": "(memberOf.contains(\"All AD Users-rshwart\"))",
              "attributeName": "sAMAccountName",
              "sourceName": "Active Directory"
            },
            "type": "accountAttribute"
          },
          "FALSE"
        ]
      },
      "type": "firstValid"
    },
    "table": {
      "FALSE": "FALSE",
      "default": "TRUE"
    }
  },
  "id": "Contains IT Access",
  "type": "lookup"
}
```

## Notes

- Multiple accounts on the same source default to using the oldest account (by creation date) unless sorting/filtering attributes are configured.
- Null values in the primary account trigger fallback to the next account's value unless `accountReturnFirstLink` is enabled (in which case the first account's value is returned even if null).
- `accountFilter` and `accountPropertyFilter` can be combined; both apply on top of the implicit source/identity filters.
