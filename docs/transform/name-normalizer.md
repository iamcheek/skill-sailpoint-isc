# Name Normalizer

**Transform type:** `"normalizeNames"`

Standardizes name formatting by applying consistent capitalization, handling special patronymic and toponymic patterns, and normalizing generational suffixes. Commonly used for proper nouns like names, but works with any string needing standardization.

Normalization logic performed by this operation:
- Proper capitalization of strings containing spaces, hyphens, or apostrophes
- Special handling of "MC"/"MAC" patterns (converts to "Mc"/"Mac")
- Consistent casing for surnames and suffixes (e.g. "VON" → "von", "DE" → "de")
- Roman numeral suffix capitalization

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `"normalizeNames"`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should be reevaluated during nightly identity refresh. Default: `false`. |
| `attributes.input` | No | object | Specifies input data source; if omitted, uses the UI-configured source/attribute. |

## Example(s)

Static value normalization — Input: `"jOHN VON SmITh"` → Output: `"John von Smith"`:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "value": "jOHN VON SmITh"
      },
      "type": "static"
    }
  },
  "type": "normalizeNames",
  "name": "Name Normalizer Transform"
}
```

Account attribute normalization — Input: `"Dr. JOHN D. O'BRIEN"` → Output: `"Dr. John D. O'Brien"`:
```json
{
  "attributes": {
    "input": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "LastName"
      },
      "type": "accountAttribute"
    }
  },
  "type": "normalizeNames",
  "name": "Name Normalizer Transform"
}
```

## Notes

No specific limitations or caveats are documented on the source page.
