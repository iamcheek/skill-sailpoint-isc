# Random Numeric

**Transform type:** `"randomNumeric"`

Generates a random number of a specified length ("generate a random number of any length").

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"randomNumeric"`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |
| `attributes.length` | No | string (integer) | Number of digits in the random output. Must be a positive number. Default: `10`. Maximum: `450` characters. |

## Example(s)

Default length (10 digits) — produces output like `"2334776774"`:
```json
{
  "type": "randomNumeric",
  "name": "Random Numeric Transform"
}
```

Custom length (6 digits) — produces output like `"759931"`:
```json
{
  "attributes": {
    "length": "6"
  },
  "type": "randomNumeric",
  "name": "Random Numeric Transform"
}
```

## Notes

- Default output length is 10 digits when `length` is unspecified.
- Maximum supported length is 450 characters.
- `length` value must be positive.
