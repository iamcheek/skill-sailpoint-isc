# Date Math

**Transform type:** `dateMath`

Performs arithmetic and rounding operations on ISO8601-formatted timestamps. It supports adding, subtracting, and rounding date/time components, and can operate on either a provided date value or the current moment using the `now` keyword. Output is formatted as `yyyy-MM-dd'T'HH:mm`.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `dateMath` |
| `name` | Yes | string | The transform's display name shown in UI dropdown menus |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false` |
| `expression` | Yes (inside `attributes`) | string | Defines date/time components and operations. Uses time unit abbreviations: `y` (year), `M` (month), `w` (week), `d` (day), `h` (hour), `m` (minute), `s` (second), `now`; and operators: `+` (add), `-` (subtract), `/` (round) |
| `roundUp` | No (inside `attributes`) | boolean | Determines rounding direction — `true` rounds up (truncate then add one unit), `false` rounds down (truncate only). Default: `false` |
| `input` | No (inside `attributes`) | object | Explicitly defines input data; if omitted, uses the UI-configured source/attribute combination |

## Example(s)

Basic operation — subtract 5 days from now and round down to the day:
```json
{
  "attributes": {
    "expression": "now-5d/d",
    "roundUp": false
  },
  "type": "dateMath",
  "name": "Date Math Transform"
}
```

Nested with `dateFormat` — add 12 hours and round up to the second, using a converted input date:
```json
{
  "attributes": {
    "expression": "+12h/s",
    "roundUp": true,
    "input": {
      "attributes": {
        "input": {
          "attributes": {
            "sourceName": "HR Source",
            "attributeName": "startDate"
          },
          "type": "accountAttribute"
        },
        "inputFormat": "MMM dd yyyy, HH:mm:ss.SSS",
        "outputFormat": "ISO8601"
      },
      "type": "dateFormat"
    }
  },
  "type": "dateMath",
  "name": "Date Math Transform"
}
```

Workday integration — add 1 day to a hire date sourced from Workday, then reformat to EPOCH_TIME_WIN32:
```json
{
  "type": "dateFormat",
  "name": "WD - HireDate",
  "attributes": {
    "input": {
      "attributes": {
        "expression": "+1d",
        "input": {
          "attributes": {
            "input": {
              "attributes": {
                "attributeName": "HIREDATE",
                "sourceName": "Workday"
              },
              "type": "accountAttribute"
            },
            "inputFormat": "MM/dd/yyyy",
            "outputFormat": "ISO8601"
          },
          "type": "dateFormat"
        },
        "roundUp": true
      },
      "type": "dateMath"
    },
    "inputFormat": "yyyy-MM-dd'T'HH:mm",
    "outputFormat": "EPOCH_TIME_WIN32"
  }
}
```

## Notes

- Input datetime must be ISO8601 format in UTC: `yyyy-MM-dd'T'HH:mm:ss.SSSZ`.
- "Week" (`w`) is unsupported as a rounding unit.
- When both `now` and an input date are specified in the expression, the transform prioritizes `now`.
- When used inside other transforms (e.g., `dateCompare`), convert the output to ISO8601 first.
