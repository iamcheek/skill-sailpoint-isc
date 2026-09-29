# Managing SailPoint ISC Roles

Use the Roles API when reading or updating role definitions, including standard membership criteria.

## Role endpoint and methods

The role-by-ID endpoint is:

```text
/roles/v1/{id}
```

The endpoint supports:

- `GET` — read a role.
- `PATCH` — update a role using JSON Patch.
- `DELETE` — delete a role.

Do not use a regular JSON merge body with `PATCH`, and do not assume `PUT` is a supported role update method. The role PATCH request body must be a JSON Patch array and use this content type:

```text
Content-Type: application/json-patch+json
```

## Permissions

The Roles API requires a token with API, `ORG_ADMIN`, `ROLE_ADMIN`, or `ROLE_SUBADMIN` authority. The user-authenticated PATCH operation requires one of these scopes:

```text
idn:role-unchecked:manage
idn:role-checked:manage
```

A `ROLE_SUBADMIN` can update a role only when all Access Profiles in the role are associated with Sources whose management workgroups include that subadmin.

## Updating membership criteria

The membership field is patchable. A standard membership criterion uses this structure:

```json
{
  "type": "STANDARD",
  "criteria": {
    "operation": "OR",
    "key": null,
    "values": null,
    "stringValue": null,
    "children": [
      {
        "operation": "EQUALS",
        "key": {
          "type": "ENTITLEMENT",
          "property": "attribute.ProfileId",
          "sourceId": "<source-id>"
        },
        "values": [
          "<profile-id-1>",
          "<profile-id-2>"
        ],
        "stringValue": null,
        "children": null
      }
    ]
  },
  "identities": null
}
```

To replace a role's full membership selector, create a JSON Patch array:

```json
[
  {
    "op": "replace",
    "path": "/membership",
    "value": {
      "type": "STANDARD",
      "criteria": {
        "operation": "OR",
        "key": null,
        "values": null,
        "stringValue": null,
        "children": [
          {
            "operation": "EQUALS",
            "key": {
              "type": "ENTITLEMENT",
              "property": "attribute.ProfileId",
              "sourceId": "<source-id>"
            },
            "values": ["<profile-id-1>", "<profile-id-2>"],
            "stringValue": null,
            "children": null
          }
        ]
      },
      "identities": null
    }
  }
]
```

Invoke it with an explicitly selected environment and a body file:

```bash
sail --env <name> api patch \
  '/roles/v1/<role-id>' \
  -H 'Content-Type: application/json-patch+json' \
  --body-file /tmp/role-membership-patch.json \
  --pretty
```

For a single additional criterion, the specification also supports JSON Patch operations such as:

```json
[
  {
    "op": "add",
    "path": "/membership/criteria/children/-",
    "value": {
      "operation": "EQUALS",
      "key": {
        "type": "ENTITLEMENT",
        "property": "attribute.memberOf",
        "sourceId": "<source-id>"
      },
      "stringValue": "Engineering"
    }
  }
]
```

Use `replace /membership` when the complete intended list is known and should be authoritative. Use `add /membership/criteria/children/-` only when extending an existing criteria tree without replacing it.

## Safe role-update workflow

1. Read the role with `GET /roles/v1/{id}` and record its current name, membership criteria, entitlements, enabled state, and other settings.
2. Confirm the target environment, role ID, source ID, and exact intended criteria values.
3. Build a JSON Patch array. Do not send a plain JSON object.
4. Use `Content-Type: application/json-patch+json`.
5. Patch only the intended field. Do not replace the entire role unless the complete role body is intentionally being managed.
6. Treat HTTP 200 as preliminary success only.
7. Re-read the role and verify the returned membership criteria, role ID/name, and unchanged fields that were outside the patch.
8. If the API returns 403, the token can likely read the role but lacks the required role-management authority or scope; do not retry with a different method or broaden the request.

The API specification lists these fields as patchable: `name`, `description`, `enabled`, `owner`, `additionalOwners`, `accessProfiles`, `entitlements`, `membership`, `requestable`, `accessRequestConfig`, `revokeRequestConfig`, `segments`, and `accessModelMetadata`.
