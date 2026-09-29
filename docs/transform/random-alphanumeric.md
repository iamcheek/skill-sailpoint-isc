# Random Alphanumeric

**Transform type:** `"randomAlphaNumeric"`

Generates a random string containing both numbers and letters (lowercase and uppercase), with configurable length up to 450 characters.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"randomAlphaNumeric"`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |
| `attributes.length` | No | string (integer) | Character count for the generated string. Must be a positive number. Default: `32`. Maximum: `450`. |

## Example(s)

Default output (32 characters):
```json
{
  "type": "randomAlphaNumeric",
  "name": "Random Alphanumeric Transform"
}
```

Custom length — 10-character output:
```json
{
  "attributes": {
    "length": "10"
  },
  "type": "randomAlphaNumeric",
  "name": "Random Alphanumeric Transform"
}
```

## Notes

- Maximum length is capped at 450 characters.
- `length` value must be positive.
- Default output length is 32 characters when `length` is unspecified.
