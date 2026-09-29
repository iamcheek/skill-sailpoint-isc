# Substring

**Transform type:** `"substring"`

The substring operation extracts a portion of a string by specifying start and end positions. It can retrieve the first n characters or extract characters from the middle of a string, using zero-based indexing.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `begin` | Yes | integer | The starting index position for the substring. If set to `-1`, begins at character `0`. |
| `beginOffset` | No | integer | Number of characters to add to the `begin` value. Only used when `begin` is not `-1`. |
| `end` | No | integer | The ending index position (exclusive). If `-1` or omitted, returns through the end of the string. |
| `endOffset` | No | integer | Number of characters to add to the `end` value. Only used when `end` is provided and not `-1`. |
| `input` | No | object | Explicitly defines the input data passed into the transform; if omitted, uses the source and attribute configuration from the UI. |

Top-level (outside `attributes`):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `substring`. |
| `name` | Yes | string | Display name in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false`. |

## Example(s)

Example 1 — basic begin/end (input `"abcdef"` → output `"cd"`):
```json
{
  "attributes": {
    "begin": 2,
    "end": 4
  },
  "type": "substring",
  "name": "Substring Transform"
}
```

Example 2 — with offsets (input `"abcdef"`, `begin: 1`, `end: 3`, `beginOffset: 1`, `endOffset: 2` → output `"cde"`):
```json
{
  "attributes": {
    "begin": 1,
    "end": 3,
    "beginOffset": 1,
    "endOffset": 2
  },
  "type": "substring",
  "name": "Substring Transform"
}
```

## Notes

- The transform lacks built-in functionality for extracting the last n characters of a string; use the "Get End of String" transform instead for that use case.
