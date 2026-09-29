# Identity Attribute

**Transform type:** `"identityAttribute"`

Retrieves the value of a user's identity attribute by system name. It is commonly used within source account creation or account disable profiles to pull identity-level data into an account attribute calculation.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `name` | Yes | string | The system name of the identity attribute to retrieve, in camel-case format (e.g. `uid`, `identificationNumber`). |
| `input` | No | object | Explicitly defines the input data passed into the transform. If omitted, the transform uses the input from the source and attribute combination configured in the UI. |

Top-level transform properties (outside the `attributes` object):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `identityAttribute`. |
| `name` | Yes | string | Display name for the transform as shown in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate the transform logic during nightly identity refresh. Default: `false`. |

## Example(s)

Retrieve the SailPoint User Name:
```json
{
  "attributes": {
    "name": "uid"
  },
  "type": "identityAttribute",
  "name": "Identity Attribute Transform"
}
```

Retrieve the employee number:
```json
{
  "attributes": {
    "name": "identificationNumber"
  },
  "type": "identityAttribute",
  "name": "Identity Attribute Transform"
}
```

## Notes

- **This transform is not intended for use within another identity profile attribute's calculation.** Because identity attribute calculation is multi-threaded, there is no guarantee that the referenced attribute has already been calculated or holds current data at the time this transform runs. Use it for source account attribute (create/disable profile) calculations instead.
