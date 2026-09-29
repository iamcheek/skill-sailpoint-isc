# Get End of String

**Transform type:** `rule`

Retrieves the rightmost N characters from a string. Implemented via the built-in `Cloud Services Deployment Utility` rule with `operation` set to `getEndOfString`. Returns `null` if the requested character count exceeds the string length.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `rule` |
| `name` | Yes | string | Transform name as displayed in UI dropdown menus |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate logic during nightly identity refresh. Default: `false` |
| `name` (inside `attributes`) | Yes | string | Must be set to `"Cloud Services Deployment Utility"` |
| `operation` (inside `attributes`) | Yes | string | Must be set to `"getEndOfString"` |
| `numChars` (inside `attributes`) | Yes | string | Number of rightmost characters to extract |
| `input` (inside `attributes`) | No | object | Explicitly defines input data; if omitted, uses the source/attribute configured in the UI |

## Example(s)

Basic usage. Input `"abcd1234"`, `numChars: "4"` → Output `"1234"`:
```json
{
  "attributes": {
    "name": "Cloud Services Deployment Utility",
    "operation": "getEndOfString",
    "numChars": "4"
  },
  "type": "rule",
  "name": "Get End Of String Transform"
}
```

Requested length exceeds string length. Input `"This is a test."` (15 characters), `numChars: "16"` → Output `null`:
```json
{
  "attributes": {
    "name": "Cloud Services Deployment Utility",
    "operation": "getEndOfString",
    "numChars": "16",
    "input": "This is a test."
  },
  "type": "rule",
  "name": "Get End Of String Transform"
}
```

## Notes

- Returns `null` when `numChars` exceeds the actual length of the input string.
- The `input` attribute is optional; when omitted, the transform uses the source/attribute values configured elsewhere (e.g. via the UI).
