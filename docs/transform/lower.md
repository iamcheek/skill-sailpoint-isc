# Lower

**Transform type:** `"lower"`

Converts an input string to all lowercase letters.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"lower"`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |
| `attributes.input` | No | object | Explicitly defines input data passed to the transform. If omitted, the transform uses input from the source and attribute combination configured in the UI. |

## Example(s)

Converts `"ACTIVE"` to `"active"`:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "ACTIVE"
      },
      "type": "static"
    }
  },
  "type": "lower",
  "name": "Lower Transform"
}
```

Converts `"All-Access"` to `"all-access"`:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "All-Access"
      },
      "type": "static"
    }
  },
  "type": "lower",
  "name": "Lower Transform"
}
```

## Notes

Minimal configuration is required; no specific limitations or caveats beyond the standard `requiresPeriodicRefresh` and `input` behavior are documented.
