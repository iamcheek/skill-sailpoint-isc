# Reference

**Transform type:** `"reference"`

Enables reuse of previously created transforms within other transforms. This allows a single transform definition to be maintained and leveraged across multiple implementations, promoting consistency and reducing duplication.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"reference"`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |
| `attributes.id` | Yes | string | Name of the pre-existing transform to reference. |
| `attributes.input` | No | object | Explicit input data for the transform; uses the configured source/attribute if omitted. |

## Example(s)

Basic reference to a previously created transform named "Get Worker Type":
```json
{
  "attributes": {
    "id": "Get Worker Type"
  },
  "type": "reference",
  "name": "Reference Transform"
}
```

Reference transforms nested inside a `concat` transform:
```json
{
  "attributes": {
    "values": [
      {
        "attributes": {
          "id": "Get Display Name"
        },
        "type": "reference"
      },
      " - ",
      {
        "attributes": {
          "id": "Get Worker Type"
        },
        "type": "reference"
      }
    ]
  },
  "type": "concat",
  "name": "Reference Transform"
}
```

## Notes

No specific limitations or caveats are explicitly documented beyond standard usage patterns. Useful for composing other transforms (e.g. `concat`) out of previously defined, reusable transform building blocks.
