# Get Reference Identity Attribute

**Transform type:** `"rule"`

This transform retrieves an identity attribute from another identity's profile (a "reference" identity) rather than from the identity currently being calculated. It is implemented as an out-of-the-box rule provided through SailPoint's Cloud Services Deployment Utility, and supports the special keyword `"manager"` as a convenient shortcut for looking up the current identity's manager.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `name` | Yes | string | Must be set to `"Cloud Services Deployment Utility"` — identifies the underlying out-of-the-box rule that implements this operation. |
| `operation` | Yes | string | Must be set to `"getReferenceIdentityAttribute"`. |
| `uid` | Yes | string | The SailPoint User Name (`uid`) of the identity whose attribute should be retrieved. Accepts the keyword `"manager"` to reference the current identity's manager, or a specific uid value to reference any other identity. |
| `attributeName` | Yes | string | The name of the identity attribute to retrieve from the referenced identity. |

Top-level transform properties (outside the `attributes` object):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be `"rule"`. |
| `name` | Yes | string | Display name for the transform as shown in UI dropdowns. |
| `requiresPeriodicRefresh` | No | boolean | Whether the transform logic should reevaluate during nightly identity refresh. Default: `false`. |

## Example(s)

Manager's email address:
```json
{
  "attributes": {
    "name": "Cloud Services Deployment Utility",
    "operation": "getReferenceIdentityAttribute",
    "uid": "manager",
    "attributeName": "email"
  },
  "type": "rule",
  "name": "Get Reference Identity Attribute Transform"
}
```

Specific user's phone attribute:
```json
{
  "attributes": {
    "name": "Cloud Services Deployment Utility",
    "operation": "getReferenceIdentityAttribute",
    "uid": "corporate.admin",
    "attributeName": "phone"
  },
  "type": "rule",
  "name": "Get Reference Identity Attribute Transform"
}
```

## Notes

- This is an "out-of-the-box rule" delivered via the Cloud Services Deployment Utility, not a dedicated native transform type — hence the top-level `type` is `"rule"` rather than a name like `getReferenceIdentityAttribute`.
- The `"manager"` keyword is a convenience shortcut for referential lookups against the calculated identity's manager; any other value in `uid` is treated as a literal SailPoint User Name.
