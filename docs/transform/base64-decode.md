# Base64 Decode

**Transform type:** `base64Decode`

Decodes data that has been encoded using Base64 text encoding, rendering it in its original binary format. Commonly used for data (such as images) that may be unsupported or damaged during transfer, storage, or output when left in text form.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `base64Decode`. |
| `name` | Yes | string | Display name shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate this transform's logic during the nightly identity refresh. Default: `false`. |

This operation has no additional input attributes beyond the top-level properties — it operates directly on the incoming attribute data configured in the identity profile attribute UI.

## Example(s)

Basic configuration:
```json
{
  "type": "base64Decode",
  "name": "Base64 Decode Transform"
}
```

String decoding example — input `"MTIzNA=="` decodes to output `1234`.

Image decoding example — input `iVBORw0KGgoAAAANSUhEUgAACUsAAAMPCAMAAADR/Oa6...` (Base64-encoded image data) decodes and renders as binary image data.

## Notes

- If the input is null, the transform returns null.
