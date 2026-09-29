# First Valid

**Transform type:** `firstValid`

Performs conditional logic across multiple data sources, returning the first non-null value encountered. It's particularly useful for required identity attributes (like usernames) where preferred data may not yet be available. The transform evaluates entries in array order, so items should be ordered by preference.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `firstValid` |
| `name` | Yes | string | Display name shown in UI dropdown menus |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false` |
| `values` | Yes (inside `attributes`) | array | Collection of attributes or nested transforms to evaluate, in precedence order |
| `ignoreErrors` | No (inside `attributes`) | boolean | If `true`, proceeds to the next option when errors (like an NPE) occur instead of halting. Default: `false` |

## Example(s)

Account attribute fallback chain — tries Active Directory, then Okta, then HR Source:
```json
{
  "attributes": {
    "values": [
      {
        "attributes": {
          "sourceName": "Active Directory",
          "attributeName": "sAMAccountName"
        },
        "type": "accountAttribute"
      },
      {
        "attributes": {
          "sourceName": "Okta",
          "attributeName": "login"
        },
        "type": "accountAttribute"
      },
      {
        "attributes": {
          "sourceName": "HR Source",
          "attributeName": "employeeID"
        },
        "type": "accountAttribute"
      }
    ]
  },
  "type": "firstValid",
  "name": "First Valid Transform"
}
```

Static fallback for a required field — falls back to the literal string `"none"` if AD `mail` is null:
```json
{
  "attributes": {
    "values": [
      {
        "attributes": {
          "sourceName": "Active Directory",
          "attributeName": "mail"
        },
        "type": "accountAttribute"
      },
      {
        "attributes": {
          "value": "none"
        },
        "type": "static"
      }
    ]
  },
  "type": "firstValid",
  "name": "First Valid Transform"
}
```

Manager DN with error handling — ignores errors (e.g. NPE when identity has no manager) and falls through to an empty string:
```json
{
  "attributes": {
    "ignoreErrors": "true",
    "values": [
      {
        "attributes": {
          "value": "$identity.manager.attributes.networkDn"
        },
        "type": "static"
      },
      ""
    ]
  },
  "name": "Example_Transform_ManagerDN",
  "type": "firstValid"
}
```

## Notes

- Returns the first non-null entry from the `values` array.
- When `ignoreErrors` is `false` (default), errors such as null pointer exceptions halt execution.
- When `ignoreErrors` is `true`, errors encountered evaluating a value are skipped and evaluation continues to the next entry in the array.
