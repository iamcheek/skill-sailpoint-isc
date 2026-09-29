# Generate Random String

**Transform type:** `rule`

Generates random strings of configurable length, with optional inclusion of numbers and special characters. Implemented via the built-in `Cloud Services Deployment Utility` rule with `operation` set to `generateRandomString`. Commonly used for creating random passwords meeting basic complexity requirements.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `rule` |
| `name` | Yes | string | Display name shown in UI dropdown menus (the transform's own display name, distinct from the `attributes.name` below) |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false` |
| `name` (inside `attributes`) | Yes | string | Must be set to `"Cloud Services Deployment Utility"` |
| `operation` (inside `attributes`) | Yes | string | Must be set to `"generateRandomString"` |
| `length` (inside `attributes`) | Yes | string | Length of the generated string. Maximum 450 characters |
| `includeNumbers` (inside `attributes`) | Yes | string | `"true"` or `"false"` — whether to include numeric characters |
| `includeSpecialChars` (inside `attributes`) | Yes | string | `"true"` or `"false"` — whether to include special characters: `!, @, #, $, %, &, *, (, ), +, <, >, ?` |

## Example(s)

16-character string with numbers and special characters:
```json
{
  "attributes": {
    "name": "Cloud Services Deployment Utility",
    "operation": "generateRandomString",
    "includeNumbers": "true",
    "includeSpecialChars": "true",
    "length": "16"
  },
  "type": "rule",
  "name": "Generate Random String Transform"
}
```

8-character string with numbers but no special characters:
```json
{
  "attributes": {
    "name": "Cloud Services Deployment Utility",
    "operation": "generateRandomString",
    "includeNumbers": "true",
    "includeSpecialChars": "false",
    "length": "8"
  },
  "type": "rule",
  "name": "Generate Random String Transform"
}
```

## Notes

- Maximum length is 450 characters.
- The docs note that "random numeric" and "random alphanumeric" transforms are recommended in most cases; this transform's primary advantage over those is support for special characters.
- Commonly used for generating passwords with basic complexity requirements.
