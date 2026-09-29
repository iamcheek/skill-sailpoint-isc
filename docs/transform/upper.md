# Upper

**Transform type:** `"upper"`

Converts an input string into all uppercase letters.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `input` | No | object | Explicitly defines the input data passed to the transform. If omitted, the transform uses the source/attribute combination configured in the UI. |

### Top-level transform properties

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"upper"`. |
| `name` | Yes | string | The transform's display name, as it appears in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |

## Example(s)

Input `"inactive"` produces output `"INACTIVE"`:

```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "inactive"
      },
      "type": "static"
    }
  },
  "type": "upper",
  "name": "Upper Transform"
}
```

Input `"Everyone"` produces output `"EVERYONE"`:

```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "Everyone"
      },
      "type": "static"
    }
  },
  "type": "upper",
  "name": "Upper Transform"
}
```

## Notes

No specific limitations or caveats are mentioned in the documentation.
