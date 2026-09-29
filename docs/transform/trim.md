# Trim

**Transform type:** `"trim"`

Removes whitespace from the beginning and end of an input string.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `input` | No | object | Explicitly defines the input data passed to the transform. If omitted, the transform uses the source/attribute combination configured in the UI. |

### Top-level transform properties

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"trim"`. |
| `name` | Yes | string | The display name for the transform, shown in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |

## Example(s)

Leading whitespace removal — input `" Vice President"` produces output `"Vice President"`:

```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": " Vice President"
      },
      "type": "static"
    }
  },
  "type": "trim",
  "name": "Trim Transform"
}
```

Trailing whitespace removal — input `"Austin, Texas "` produces output `"Austin, Texas"`:

```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "Austin, Texas "
      },
      "type": "static"
    }
  },
  "type": "trim",
  "name": "Trim Transform"
}
```

## Notes

No limitations or caveats are explicitly documented for this operation.
