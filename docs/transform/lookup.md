# Lookup

**Transform type:** `"lookup"`

The lookup transform accepts an incoming string value and compares it against a collection of key-value pairs. When the input matches a key, the corresponding value is returned. If no match exists and a `"default"` key is configured in the table, its value is returned; otherwise an error occurs.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"lookup"`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |
| `attributes.table` | Yes | object | A JSON object mapping keys to values. One key must be named `"default"` to provide a fallback output when the input matches no other keys. |
| `attributes.input` | No | object | Explicitly defines input data. If omitted, the transform uses the source/attribute combination configured in the UI. |

Note: duplicate key names inside `table` are not permitted and will throw an error. Multiple keys may map to identical values, but each key itself must be unique. Omitting the `"default"` key causes an error if the input matches no other keys.

## Example(s)

Area code to city mapping, with a default fallback:
```json
{
  "attributes": {
    "table": {
      "512": "Austin",
      "281": "Houston",
      "214": "Dallas",
      "210": "San Antonio",
      "default": "Unknown Area"
    }
  },
  "type": "lookup",
  "name": "Lookup Transform"
}
```

Invalid configuration — duplicate keys (`"713"` and `"832"` both map to `"Houston"`, which is fine, but this illustrates the shape prior to noting duplicate *keys* are disallowed; keys here are all unique area codes, values may repeat):
```json
{
  "attributes": {
    "table": {
      "512": "Austin",
      "281": "Houston",
      "713": "Houston",
      "832": "Houston",
      "214": "Dallas",
      "210": "San Antonio"
    }
  },
  "type": "lookup",
  "name": "Test Lookup Transform"
}
```

## Notes

- A `"default"` key in `table` is mandatory to avoid errors when no other key matches.
- Duplicate *keys* in `table` are not permitted and will cause an error; duplicate *values* mapped from different keys are allowed.
