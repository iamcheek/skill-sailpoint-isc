# Join

**Transform type:** `"join"`

Combines two or more string values into a single output using a specified separator. A common use is merging first and last name attributes into a full display name, but it supports any concatenation scenario where multiple values (static strings or nested transform outputs) need to be combined.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `values` | Yes | array | The items to combine. Each item can be a static string or the output of a nested transform (e.g. an `accountAttribute` transform). |
| `separator` | No | string | The string inserted between joined items. Default: `","`. |

Top-level transform properties (outside the `attributes` object):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be `"join"`. |
| `name` | Yes | string | Display name for the transform in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate logic during nightly identity refresh. Default: `false`. |

## Example(s)

Combining first and last names with a space separator:
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
      {
        "attributes": {
          "sourceName": "HR Source",
          "attributeName": "LastName"
        },
        "type": "accountAttribute"
      }
    ],
    "separator": " "
  },
  "type": "join",
  "name": "Join Transform"
}
```

Joining job title with job code using a hyphen separator:
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
      {
        "attributes": {
          "sourceName": "HR Source",
          "attributeName": "JobCode"
        },
        "type": "accountAttribute"
      }
    ],
    "separator": "-"
  },
  "type": "join",
  "name": "Join Transform"
}
```
