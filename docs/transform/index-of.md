# Index Of

**Transform type:** `"indexOf"`

Locates the position of the first occurrence of a specific substring within the incoming data, returning the numeric index as a string. Commonly paired with the Substring transform to extract dynamic-length portions of a string.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `substring` | Yes | string | The substring to search for within the incoming data. Returns `-1` if the substring is not found. |
| `input` | No | object | Explicitly defines the input data passed into the transform. If not provided, uses input from the source and attribute combination configured in the UI. |

Top-level transform properties (outside the `attributes` object):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"indexOf"`. |
| `name` | Yes | string | The transform name as displayed in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate logic during nightly identity refresh. Default: `false`. |

## Example(s)

Find `"admin_"` in `"admin_jsmith"` (returns `"0"`):
```json
{
  "attributes": {
    "substring": "admin_"
  },
  "type": "indexOf",
  "name": "Index Of Transform"
}
```

Find `"b"` in `"abcabcabc"` (returns `"1"`, the first occurrence):
```json
{
  "attributes": {
    "substring": "b"
  },
  "type": "indexOf",
  "name": "Index Of Transform"
}
```

## Notes

- Returns the position of the **first occurrence only**.
- Use the **Last Index Of** transform to find the last occurrence instead.
- Use the **Substring** transform in conjunction with this one to extract non-first/non-last occurrences or dynamic-length substrings.
- Returns `-1` if the substring is not found in the incoming data.
