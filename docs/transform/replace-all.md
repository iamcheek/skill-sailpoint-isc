# Replace All

**Transform type:** `"replaceAll"`

Performs multiple pattern replacements on incoming data. Unlike the basic `replace` operation, it can find and replace multiple character patterns simultaneously using standard regex syntax.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"replaceAll"`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |
| `attributes.table` | Yes | object | Key-value pairs where keys are regex patterns to find and values are their replacement strings. Supports standard regex syntax. If missing or empty, the transform returns the original value unchanged (no longer errors in this case). |
| `attributes.input` | No | object | Explicitly defines input data; if omitted, uses the source and attribute combination configured in the UI. |

## Example(s)

Special character replacements — Input: `"Enrique Jose-Piñon"` → Output: `"Enrique Jose Pinon"`:
```json
{
  "attributes": {
    "table": {
      ".": "-",
      "\"": "'",
      "ñ": "n"
    }
  },
  "type": "replaceAll",
  "name": "Replace All Transform"
}
```

Complex regex patterns — Input: `"ad512.777.1234"` → Output: `"512-777-1234"`:
```json
{
  "attributes": {
    "input": "",
    "table": {
      "[.]": "-",
      "[a-zA-z]": ""
    }
  },
  "type": "replaceAll",
  "name": "Replace All Transform"
}
```

## Notes

- The transform no longer errors when the `table` attribute is missing or empty — it returns the original value unchanged in that case.
- Supports standard regex syntax patterns in `table` keys.
