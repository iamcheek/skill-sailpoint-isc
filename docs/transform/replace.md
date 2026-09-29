# Replace

**Transform type:** `"replace"`

The replace operation locates a specified character pattern within incoming data and substitutes all occurrences with an alternate value, using standard regex syntax.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `regex` | Yes | string | The pattern to search for and replace, using standard regex syntax. |
| `replacement` | Yes | string | The replacement string that substitutes the pattern wherever it occurs. |
| `input` | No | object | Explicitly defines the input data passed into the transform; if omitted, uses the source and attribute configuration from the UI. |

Top-level (outside `attributes`):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `replace`. |
| `name` | Yes | string | The transform name as displayed in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate transform logic during evening identity refresh. Default: `false`. |

## Example(s)

Configuration structure:
```json
{
  "attributes": {
    "regex": "pattern",
    "replacement": "newValue"
  },
  "type": "replace",
  "name": "Transform Name"
}
```

Example 1 — simple word replacement (input `"Working with IIQ is fun"` → output `"Working with Identity Security Cloud is fun"`):
```json
{
  "attributes": {
    "regex": "IIQ",
    "replacement": "Identity Security Cloud"
  },
  "type": "replace",
  "name": "Replace Transform"
}
```

Example 2 — regex pattern for character removal (output `"Thequickbrownfoxjumpedoverlazydogs"`):
```json
{
  "attributes": {
    "input": "The quick brown fox jumped over 10 lazy dogs",
    "regex": "[^a-zA-Z]",
    "replacement": ""
  },
  "type": "replace",
  "name": "Replace Transform"
}
```

## Notes

- Supports standard regex syntax for pattern matching.
- Reference materials suggested by the docs for building regex patterns: regular-expressions.info, regex101.com, and extendsclass.com regex tester.
