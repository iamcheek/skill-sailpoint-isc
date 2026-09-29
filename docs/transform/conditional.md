# Conditional

**Transform type:** `conditional`

Evaluates a comparison expression and outputs one of two values depending on the result. Provides a simpler alternative to the static transform, with null-safe error checking and straightforward conditional logic.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"conditional"`. |
| `name` | Yes | string | Display name in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate this transform's logic during the nightly identity refresh. Default: `false`. |
| `expression` | Yes | string | A comparison in the format `"ValueA eq ValueB"`. Both values can be static strings or transform outputs referenced via `$variableName`. Only the `eq` operator is supported. |
| `positiveCondition` | Yes | string | The output returned when `expression` evaluates to true. Can reference a `$variableName` for a nested transform's output. |
| `negativeCondition` | Yes | string | The output returned when `expression` evaluates to false. Can reference a `$variableName` for a nested transform's output. |
| *(custom properties)* | No | any | Additional custom-named properties (e.g. `department`, `scienceBuilding`) act as variables referenced via `$variableName` syntax inside `expression`, `positiveCondition`, or `negativeCondition`. These can contain nested transform definitions (e.g. `accountAttribute`, `static`). |

## Example(s)

Example 1 — Basic department check:
```json
{
  "attributes": {
    "expression": "$department eq Science",
    "positiveCondition": "true",
    "negativeCondition": "false",
    "department": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "department"
      },
      "type": "accountAttribute"
    }
  },
  "type": "conditional",
  "name": "Test Conditional Transform"
}
```

Example 2 — Using nested transforms as the positive/negative outputs:
```json
{
  "attributes": {
    "expression": "$department eq Science",
    "positiveCondition": "$scienceBuilding",
    "negativeCondition": "$adminBuilding",
    "department": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "department"
      },
      "type": "accountAttribute"
    },
    "scienceBuilding": {
      "attributes": {
        "value": "Building S"
      },
      "type": "static"
    },
    "adminBuilding": {
      "attributes": {
        "value": "Building A"
      },
      "type": "static"
    }
  },
  "type": "conditional",
  "name": "Test Conditional Transform"
}
```

## Notes

- Operands cannot be null; null values trigger an `IllegalArgumentException`.
- Only the `eq` operator is valid in `expression`; any other operator causes an `IllegalArgumentException`.
- String comparisons are case-sensitive (e.g. `"engineering"` does not equal `"Engineering"`).
