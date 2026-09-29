# Date Compare

**Transform type:** `dateCompare`

Evaluates a comparison between two dates and returns one of two values based on the result. Commonly used for lifecycle state calculations (e.g., determining if a user is "active" based on hire/termination dates).

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"dateCompare"`. |
| `name` | Yes | string | Display name in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate this transform's logic during the nightly identity refresh. Default: `false`. |
| `firstDate` | Yes | string or object | The left-hand date value. Must be in ISO8601 format, the literal keyword `"now"`, or a nested transform (e.g. `accountAttribute`, `dateFormat`) that produces a date value. |
| `secondDate` | Yes | string or object | The right-hand date value. Same format rules as `firstDate` (ISO8601, `"now"`, or a nested transform). |
| `operator` | Yes | string | Comparison operator. Valid values: `LT`, `LTE`, `GT`, `GTE` (shown in examples in lowercase form as `gt`/`lte`). |
| `positiveCondition` | Yes | string | Value returned when the comparison evaluates true. |
| `negativeCondition` | Yes | string | Value returned when the comparison evaluates false. |

## Example(s)

Example 1 — Lifecycle state based on termination date vs. now:
```json
{
  "attributes": {
    "firstDate": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "termination_date"
      },
      "type": "accountAttribute"
    },
    "secondDate": "now",
    "operator": "gt",
    "positiveCondition": "active",
    "negativeCondition": "terminated"
  },
  "type": "dateCompare",
  "name": "Date Compare Transform"
}
```

Example 2 — Legacy user classification comparing hire date to a fixed formatted date:
```json
{
  "attributes": {
    "firstDate": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "hire_date"
      },
      "type": "accountAttribute"
    },
    "secondDate": {
      "attributes": {
        "input": "12/31/1995",
        "inputFormat": "M/d/yyyy",
        "outputFormat": "ISO8601"
      },
      "type": "dateFormat"
    },
    "operator": "lte",
    "positiveCondition": "legacy",
    "negativeCondition": "regular"
  },
  "type": "dateCompare",
  "name": "Date Compare Transform"
}
```

## Notes

- Both date inputs must adhere to ISO8601 formatting standards (unless using the `"now"` keyword or a nested transform that produces an ISO8601 value).
- The transform can nest other transform types (such as `accountAttribute` or `dateFormat`) as the value of `firstDate` or `secondDate`.
- No default values are specified for any attribute; all required fields must be explicitly provided.
