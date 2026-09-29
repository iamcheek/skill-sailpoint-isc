# UUID Generator

**Transform type:** `"uuid"`

Creates a universally unique identifier (UUID) as a 36-character random string. It generates random UUIDs without performing an active uniqueness check, though collisions are statistically extremely unlikely (approximately a 1-in-68.7-billion chance within a tenant).

## Attributes

The UUID generator has no separate input attributes or configuration parameters — it accepts only the top-level transform properties.

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"uuid"`. |
| `name` | Yes | string | Display name for the transform, shown in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |

## Example(s)

Basic UUID generator configuration:

```json
{
  "type": "uuid",
  "name": "UUID Generator Transform"
}
```

Example output produced by the transform:

```
"f7493c55-f3fc-491a-b352-4664d71f885b"
```

## Notes

- There is no uniqueness checking in this transform. Collisions are theoretically possible but statistically extremely improbable (documentation cites roughly a 1 in 68.7 billion chance of a collision within a tenant).
