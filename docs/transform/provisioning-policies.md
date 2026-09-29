# Transforms in provisioning policies

Provisioning-policy transforms run when ISC creates or updates an account on a source. They are configured inline on a field in the source's provisioning policy and are evaluated before the value is sent to the connector.

SailPoint's guide: [Transforms in Provisioning Policies](https://developer.sailpoint.com/docs/extensibility/transforms/guides/transforms-in-provisioning-policies/).

## Provisioning-policy API

Provisioning policies are source-scoped resources. The `usageType` identifies the policy:

| Policy | `usageType` |
|---|---|
| Create Account | `CREATE` |
| Update Account | `UPDATE` |
| Enable Account | `ENABLE` |
| Disable Account | `DISABLE` |
| Delete Account | `DELETE` |

The v3 endpoints for a policy are:

```text
GET /v3/sources/{sourceId}/provisioning-policies/{usageType}
PUT /v3/sources/{sourceId}/provisioning-policies/{usageType}
```

A policy response contains a `fields` array. Add or modify a transform on the appropriate field in that array, then submit the complete policy for an update. Reading a policy is safe; do not use `PUT` unless the user explicitly requests a change.

## Identity values during provisioning

When a provisioning-policy transform needs an identity attribute, use an `identityAttribute` transform:

```json
{
  "type": "identityAttribute",
  "attributes": {
    "name": "department"
  }
}
```

The provisioning-policy guide explicitly warns that `accountAttribute` does not work for retrieving the identity's value during provisioning. A child `identityAttribute` can be nested inside another transform's `attributes.input`.

For example, this transform preserves the identity's department value while limiting it to 255 characters:

```json
{
  "name": "Department",
  "transform": {
    "type": "substring",
    "attributes": {
      "input": {
        "type": "identityAttribute",
        "attributes": {
          "name": "department"
        }
      },
      "begin": 0,
      "end": 255
    }
  },
  "attributes": {},
  "isRequired": false,
  "type": "string",
  "isMultiValued": false
}
```

`substring` uses zero-based indexing and an exclusive `end`, so `begin: 0` and `end: 255` return at most 255 characters. The same pattern can be used for a `title` identity attribute.

## Static versus value-limiting transforms

A `static` transform returns one fixed value for every account:

```json
{
  "type": "static",
  "attributes": {
    "value": "Fixed value for every account"
  }
}
```

Do not use `static` merely to enforce a target-system character limit. It replaces the identity's value rather than truncating it. Use `substring` around an `identityAttribute` when the requirement is to preserve the value but limit its length.

The field's declared `type` must match the transform output. For example, a text field whose transform returns a string should have `"type": "string"`; do not copy an inconsistent field type from an example without checking the source schema.

## Important behavior

- These transforms run during account create/update provisioning, not during identity aggregation or refresh.
- The transform JSON is inline in the provisioning-policy field; it is not necessarily a separately-created transform definition managed with `sail transform`.
- A provisioning-policy update is a complete-policy update. Preserve all existing fields and metadata when constructing the `PUT` body.
- To validate behavior, use a controlled provisioning event for an identity with no existing account on the source. A transform preview for a separately-managed transform is not the same as testing an account create policy.
