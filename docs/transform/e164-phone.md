# E.164 Phone

**Transform type:** `e164phone`

Converts incoming phone number strings into E.164-compatible format. If the input string does not represent a valid phone number, the transform returns `null`.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `e164phone` |
| `name` | Yes | string | The transform's name as displayed in UI dropdown menus |
| `requiresPeriodicRefresh` | No | boolean | Whether transform logic should be reevaluated during nightly identity refresh. Default: `false` |
| `defaultRegion` | No (inside `attributes`) | string | Phone number region for formatting, in ISO 3166-1 alpha-2 format. Default: `US` |
| `input` | No (inside `attributes`) | object | Explicitly defines input data passed to the transform; if omitted, uses the configured source/attribute |

## Example(s)

Hyphen-separated phone number. Input `512-777-2222` → Output `+1512459222`:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "512-777-2222"
      },
      "type": "static"
    }
  },
  "type": "e164phone",
  "name": "E.164Phone Transform"
}
```

Period-separated phone number. Input `779.284.2727` → Output `+17792842727`:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "779.284.2727"
      },
      "type": "static"
    }
  },
  "type": "e164phone",
  "name": "E.164Phone Transform"
}
```

International number with explicit region. Input `0412345678` with `defaultRegion: "AU"` → Output `+61412345678`:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "0412345678"
      },
      "type": "static"
    },
    "defaultRegion": "AU"
  },
  "type": "e164phone",
  "name": "E.164Phone Transform"
}
```

## Notes

- Returns `null` when the input string does not represent a valid phone number.
- `defaultRegion` defaults to `US` when not specified, and is used to interpret numbers that don't include a country code.
