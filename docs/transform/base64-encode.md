# Base64 Encode

**Transform type:** `base64Encode`

Encodes incoming data using Base64 encoding, producing a string of 64 basic ASCII characters. Commonly used to encode binary data (like images) for representation in text documents, or to protect data that might otherwise be damaged during transfer or storage.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `base64Encode`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate this transform's logic during the nightly identity refresh. Default: `false`. |

This operation has no additional input attributes beyond the top-level properties. It accepts any data type as input (strings, binary data).

## Example(s)

Example 1 — Simple string encoding:
```json
{
  "type": "base64Encode",
  "name": "Base64 Encode Transform"
}
```
Input: `1234` → Output: `MTIzNA==`

Example 2 — Binary image encoding:
```json
{
  "type": "base64Encode",
  "name": "Base64 Encode Transform"
}
```
Input: binary image file → Output: `iVBORw0KGgoAAAANSUhEUgAACUsAAAMPCAMAAADR/Oa6AAAAM1BMVEX...` (truncated)

## Notes

- If the input to the Base64 Encode transform is null, the transform returns a null value.
