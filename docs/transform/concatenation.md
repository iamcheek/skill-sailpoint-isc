# Concatenation

**Transform type:** `concat`

Joins multiple string values into a single combined output. A common use case is combining first and last names into a full display name, though it supports any scenario requiring string joining.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"concat"`. |
| `name` | Yes | string | Display name appearing in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate this transform's logic during the nightly identity refresh. Default: `false`. |
| `values` | Yes | array | An array of items to join. Items can be static strings or return values from nested transforms. |

## Example(s)

Example 1 — Concatenate first name, a space, last name, and a static suffix:
```json
{
  "attributes": {
    "values": [
      {
        "attributes": {
          "sourceName": "HR Source",
          "attributeName": "FirstName"
        },
        "type": "accountAttribute"
      },
      " ",
      {
        "attributes": {
          "sourceName": "HR Source",
          "attributeName": "LastName"
        },
        "type": "accountAttribute"
      },
      " (Contractor)"
    ]
  },
  "type": "concat",
  "name": "Test Concat Transform"
}
```

Example 2 — Concatenate job title and job code with a hyphen separator:
```json
{
  "attributes": {
    "values": [
      {
        "attributes": {
          "sourceName": "HR Source",
          "attributeName": "JobTitle"
        },
        "type": "accountAttribute"
      },
      "-",
      {
        "attributes": {
          "sourceName": "HR Source",
          "attributeName": "JobCode"
        },
        "type": "accountAttribute"
      }
    ]
  },
  "type": "concat",
  "name": "Test Concat Transform"
}
```
