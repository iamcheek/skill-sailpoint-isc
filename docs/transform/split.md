# Split

**Transform type:** `"split"`

The split transform takes an incoming string, uses a specified delimiter (a single character or regex pattern) to break it into an array, and returns a specific element from that array. Useful for decomposing combined values (e.g. full names) into parts, or extracting individual items from ordered lists.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `delimiter` | Yes | string | A single character or regex pattern marking the split points between substrings. |
| `index` | Yes | integer | Zero-based array index of the element to return after splitting. |
| `throws` | No | boolean | Whether to throw an `IndexOutOfBoundsException` when `index` exceeds the array bounds, or return `null` instead. Default: `true`. |
| `input` | No | object | Explicit string to process; if omitted, uses the configured source and attribute. |

Top-level (outside `attributes`):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `split`. |
| `name` | Yes | string | Display name for the transform in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false`. |

## Example(s)

Example 1 — basic delimiter usage (input `"abc:123"` → output `"123"`):
```json
{
  "attributes": {
    "delimiter": ":",
    "index": 1
  },
  "type": "split",
  "name": "Split Transform"
}
```

Example 2 — space delimiter with explicit input and `throws` set (input `"The quick brown fox jumped over 10 lazy dogs"` → output `"fox"`):
```json
{
  "attributes": {
    "input": "The quick brown fox jumped over 10 lazy dogs",
    "delimiter": " ",
    "index": 3,
    "throws": true
  },
  "type": "split",
  "name": "Split Transform"
}
```

## Notes

- The transform uses zero-based (0-based) indexing for array positions after splitting occurs.
