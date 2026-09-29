# Display Name

**Transform type:** `displayName`

Generates an identity's display name by prioritizing the Preferred Name field (when available) over the Given Name, then appending the Family Name. The resulting format is: `(Preferred Name or Given Name) + Family Name`.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `displayName` |
| `name` | Yes | string | Display name shown in UI dropdown menus |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false` |
| `input` | Yes | string | The input value for the transform. The `attributes` key must be present and not null. The transform automatically uses the identity's Preferred Name (if available) or Given Name, combined with the Family Name |

## Example(s)

Standard configuration:
```json
{
  "name": "Display Name Transform",
  "type": "displayName",
  "attributes": {
    "input": "input"
  }
}
```

## Notes

- Behavior with Preferred Name set: Preferred Name `"John"` + Family Name `"Doe"` = `"John Doe"`.
- Behavior without Preferred Name: Given Name `"Jonathan"` + Family Name `"Doe"` = `"Jonathan Doe"`.
- The transform operates on identity attributes (Preferred Name, Given Name, Family Name) automatically rather than on an arbitrary configured source/attribute — it is purpose-built for the preference hierarchy of Preferred Name over Given Name.
