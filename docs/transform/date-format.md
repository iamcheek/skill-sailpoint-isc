# Date Format

**Transform type:** `dateFormat`

Converts datetime strings from one format to another, supporting both Java `SimpleDateFormat` syntax and built-in named formats. Useful when syncing date/time data between systems that use different formats.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `dateFormat`. |
| `name` | Yes | string | Display name in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate this transform's logic during the nightly identity refresh. Default: `false`. |
| `inputFormat` | No | string | The format of the incoming date value. Accepts a Java `SimpleDateFormat` pattern or one of the built-in named formats: `ISO8601`, `LDAP`, `PEOPLE_SOFT`, `EPOCH_TIME_JAVA`, `EPOCH_TIME_WIN32`. Default: `ISO8601`. |
| `outputFormat` | No | string | The desired output format. Accepts a Java `SimpleDateFormat` pattern or one of the same built-in named formats. Default: `ISO8601`. |
| `input` | No | object | Explicitly defines the input data (e.g. a nested transform or literal value). If omitted, uses the source/attribute combination configured in the UI. |

## Example(s)

Example 1 — Convert a Java epoch timestamp to ISO8601:
```json
{
  "attributes": {
    "inputFormat": "EPOCH_TIME_JAVA",
    "outputFormat": "ISO8601"
  },
  "type": "dateFormat",
  "name": "Date Format Transform"
}
```

Example 2 — Convert a US-style date to a database date format:
```json
{
  "attributes": {
    "inputFormat": "M/d/yyyy",
    "outputFormat": "yyyy-MM-dd"
  },
  "type": "dateFormat",
  "name": "Date Format Transform"
}
```

## Notes

- The transform does not currently support the `"now"` keyword as an input value (unlike `dateCompare`).
- Custom (non-named) formats leverage Java `SimpleDateFormat` syntax.
