# Decompose Diacritical Marks

**Transform type:** `decomposeDiacriticalMarks`

Standardizes characters with diacritical marks (accents, umlauts, tildes, etc.) by decomposing them into base characters and removing the diacritical components — e.g. converting "Ā", "Ĉ", "Ň", "Ŵ" into ASCII-compatible equivalents. This makes data compatible with downstream systems that don't support special symbols. Internally it uses Java's `Normalizer` library with NFKD form, followed by a regex replacement to strip combining diacritical marks.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `decomposeDiacriticalMarks` |
| `name` | Yes | string | Display name shown in UI dropdown menus |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false` |
| `input` | No (inside `attributes`) | object | Explicitly defines input data; if omitted, uses the source/attribute configured in the UI |

## Example(s)

Simple example — basic decomposition (no explicit input; uses UI-configured attribute). Input `"Āric"` → Output `"Aric"`:
```json
{
  "type": "decomposeDiacriticalMarks",
  "name": "Test Decompose Diacritical Marks Transform"
}
```

Advanced example — with explicit input source. Input `"Dubçek"` → Output `"Dubcek"`:
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
  "type": "decomposeDiacriticalMarks",
  "name": "Decompose Diacritical Marks Transform"
}
```

## Notes

- Uses Unicode Normalization Form KD (NFKD) per Unicode Standard Annex 4.
- After normalization, applies the regex pattern `[\p{InCombiningDiacriticalMarks}]` to strip the diacritical marks.
- SailPoint's documentation includes a Java code snippet demonstrating equivalent logic for testing custom implementations.
