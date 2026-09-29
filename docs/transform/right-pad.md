# Right Pad

**Transform type:** `"rightPad"`

The right pad transform appends a specified character to the end of an incoming string until it reaches a desired length. This is useful for normalizing data (e.g. employee IDs) that vary in length but need consistency for downstream systems.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `length` | Yes | string | Integer value specifying the final output string length. |
| `padding` | No | string | Character used to pad the incoming data. Default: `" "` (single space). |
| `input` | No | object | Explicitly defines the input data passed into the transform; if omitted, uses the source and attribute configuration from the UI. |

Top-level (outside `attributes`):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `rightPad`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate transform during nightly identity refresh. Default: `false`. |

## Example(s)

Example 1 — basic padding with zeros (input `"1234"` → output `"12340000"`):
```json
{
  "attributes": {
    "padding": "0",
    "length": "8"
  },
  "type": "rightPad",
  "name": "Right Pad Transform"
}
```

Example 2 — with account attribute input (input `"1234"` → output `"1234xxx"`):
```json
{
  "attributes": {
    "padding": "x",
    "length": "7",
    "input": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "employeeID"
      },
      "type": "accountAttribute"
    }
  },
  "type": "rightPad",
  "name": "Right Pad Transform"
}
```

## Notes

- If the input value is `null`, the transform returns `null`.
- The padding character defaults to a single space if not specified.
