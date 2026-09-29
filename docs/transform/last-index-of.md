# Last Index Of

**Transform type:** `"lastIndexOf"`

Locates the last occurrence of a specific substring within an incoming value, returning the index position of that final occurrence, or `-1` if the substring is not found.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `substring` | Yes | string | The string whose last occurrence location within the incoming data you want to find. |
| `input` | No | object | Explicitly defines the input data passed into the transform. If not provided, the transform uses input from the source and attribute combination configured in the UI. |

Top-level transform properties (outside the `attributes` object):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `lastIndexOf`. |
| `name` | Yes | string | The display name for the transform in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false`. |

## Example(s)

Find last occurrence of `"admin_"` in `"admin_jsmith"` (returns `"0"`, only occurrence):
```json
{
  "attributes": {
    "substring": "admin_"
  },
  "type": "lastIndexOf",
  "name": "Last Index Of Transform"
}
```

Find last occurrence of `"b"` in `"abcabcabc"` (returns `"7"`):
```json
{
  "attributes": {
    "substring": "b"
  },
  "type": "lastIndexOf",
  "name": "Last Index Of Transform"
}
```

## Notes

- For the **first** occurrence instead, use the **Index Of** transform.
- When the substring occurs multiple times, only the index of the last occurrence is returned.
- Returns `-1` if the substring is not found.
