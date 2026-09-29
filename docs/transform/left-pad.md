# Left Pad

**Transform type:** `"leftPad"`

Pads an incoming string with a user-supplied character on the left side until it reaches a specified length. Commonly used for data normalization when values such as employee IDs need a uniform length for downstream systems.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `length` | Yes | string | An integer value (as a string) specifying the final output string's desired length. |
| `padding` | No | string | The character used for left-side padding. Default: a single space `" "`. |
| `input` | No | object | Explicitly defines the input data; if omitted, uses the source and attribute combination configured in the UI. |

Top-level transform properties (outside the `attributes` object):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be `leftPad`. |
| `name` | Yes | string | Transform name as displayed in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during identity refresh. Default: `false`. |

## Example(s)

Basic example — pad with zeros to length 8. Input `"1234"` → output `"00001234"`:
```json
{
  "attributes": {
    "padding": "0",
    "length": "8"
  },
  "type": "leftPad",
  "name": "Left Pad Transform"
}
```

Advanced example — with explicit nested input source, padding with `"x"` to length 7. Input `"1234"` → output `"xxx1234"`:
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
  "type": "leftPad",
  "name": "Left Pad Transform"
}
```

## Notes

- If the input is `null`, the transform returns `null`.
- The `length` attribute determines the final string size (padding is added only up to this length; it does not truncate longer input strings, per standard left-pad semantics).
