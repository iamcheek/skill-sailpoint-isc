# Web Services SaaS Connector

Connector-specific reference for sources using SailPoint's **Web Services SaaS** connector. See
[README.md](README.md) for how this connector catalog is organized and when to load a connector
reference.

This file covers general Web Services SaaS configuration patterns: schema relationships, response
mapping, parent/child operations, pagination, and aggregation verification. Check every pattern
against the source's current configuration and connector version before changing a source.

## Entitlement and group schema pattern

For a Web Services SaaS source to persist API objects as group entitlements and associate those
entitlements with accounts, configure both sides of the relationship:

1. Create a **group schema** for the resource being aggregated. Its important semantic fields are:
   - an `identityAttribute` (for example, `id`),
   - a `displayAttribute` (for example, `name`), and
   - attributes for fields returned by the API, such as a group name and multivalued membership.
2. Add an **account-schema attribute** for that group schema. The attribute must reference the
   group schema's ID and be marked as both:
   - `isEntitlement: true`, and
   - `isGroup: true`.
   It is normally multivalued when an account can belong to more than one group.
3. Add the correctly named group operation:

   ```text
   Group Aggregation-<schema-name>
   ```

The exact serialized source JSON is authoritative. Treat examples such as the following as a
shape, not as a complete source update payload:

```json
{
  "name": "groups",
  "type": "String",
  "schema": "<group-schema-id>",
  "multi": true,
  "isEntitlement": true,
  "isGroup": true
}
```

A group resource being returned by the connector is not the same thing as an entitlement being
persisted or associated to an account. Verify all three separately:

1. **Connector resource scan** — what the HTTP operation returned and what the connector reports as
   scanned.
2. **Entitlement persistence** — what an entitlement endpoint and the ISC UI can retrieve.
3. **Account association** — whether the account's entitlement attribute contains the entitlement
   identity and whether `/accounts/v1` reports the expected entitlement state.

An HTTP 200 response or a log line such as `Entitlements: N` proves connector output, not
necessarily persistence. A generic `RO processed:undefined` debug-log value is not, by itself, an
error; verify the resulting resource and account instead.

## Response roots and mappings

Web Services response mappings are sensitive to the configured response root. The connector
prefixes mapping paths with the configured root, so configure the root and relative mappings as a
pair. For a response shaped like this:

```json
{
  "results": [
    { "id": "group-1", "name": "Sales", "members": [{ "id": "user-1" }] }
  ]
}
```

A common configuration is a `$.results` root with relative mappings for `id`, `name`, and
`members[*].id`. Conceptually, the array element paths are:

```text
$.results[*].id
$.results[*].name
$.results[*].members[*].id
```

Do not accidentally include the root twice (for example, `$.results.$.results[*].id`). Conversely,
do not assume that a list response will be treated as an array of resources unless the root/array
handling is explicitly configured. Test a single response and inspect the resulting resource
object before running a full aggregation.

## Parent and child Web Services operations

Use a parent operation when its response contains the account fields needed for normal aggregation,
and use a child operation when a second endpoint is needed to enrich each parent resource. In the
Web Services operation configuration, the child refers to its parent through **Parent Endpoint**.
Connector operation variables use the parent response, for example:

```text
$response.id$
$getobject.nativeIdentity$
```

The first is useful when a child request needs the current parent resource's ID. The second is useful
when the operation needs the native identity of the account being read or provisioned. Confirm the
variable's scope in the operation UI/source JSON; these are connector operation expressions, not
ordinary JSONPath.

For APIs whose find-by-id endpoint expects a JSON array, use the array body exactly:

```json
["$response.id$"]
```

Do not change it to an object such as `{ "ids": ["$response.id$"] }` unless that endpoint actually
documents the object form. A child operation can then map only the fields that are absent from the
parent response, reducing the chance that it replaces or drops parent attributes.

## Pagination

Treat pagination as part of the source-specific API contract. Before enabling or changing an
operation, verify:

- whether page numbering is zero- or one-based;
- whether the API uses page/offset, a continuation token, or another mechanism;
- whether the response reports the actual page size and total;
- whether the ordering is deterministic across pages; and
- whether the connector's pagination script uses the response fields and syntax supported by the
  current connector version.

Prefer a documented deterministic ordering when the API supports one. An unstable order can cause
duplicates or omissions between pages; changing page size alone is not a reliable fix. Compare the
number of scanned records with the number of unique native identities after aggregation.

## Aggregation ordering and troubleshooting

When an account schema references a group schema, make the group operation name, schema reference,
and aggregation order explicit. At minimum, verify:

- the group schema ID referenced by the account attribute is the current one;
- the account attribute is multivalued and has both entitlement/group flags;
- the operation is named `Group Aggregation-<schema-name>` exactly;
- the group aggregation and account aggregation are enabled and run in the intended order;
- the response root and mappings produce resource objects with stable identities; and
- the final account resource still contains the entitlement attribute after all parent/child and
  after-operation processing.

A successful HTTP response does not rule out a later mapping or post-processing problem. For a
controlled test, aggregate one known user, inspect the parent response/resource, inspect the child
resource, and then inspect the final post-customizer User RO and persisted account. Compare each
stage rather than relying only on connector counts.

If a newly mapped account attribute is absent from the final User RO or persisted account, compare
the raw connector resource, the post-customizer User RO, and the saved account. A customizer or
after-operation step may be removing or replacing the value; establish where the value disappears
before changing the connector mappings.

## How to use this during an investigation

1. Identify the source as **Web Services SaaS** and load this file after a concrete aggregation,
   mapping, pagination, or entitlement symptom is available.
2. Re-read the current source configuration before changing anything. UI edits can remove pagination
   steps, query parameters, mappings, or child operations.
3. Separate HTTP output, connector resource objects, persisted entitlements, account associations,
   and post-customizer account resources.
4. Treat the patterns here as connector guidance, not as a complete source configuration. Verify
   operation names, enabled states, endpoint paths, response roots, mappings, and pagination against
   the current source and API.
5. If the symptom is a source reset or pending-task blocker rather than Web Services mapping or
   aggregation, use the applicable investigation playbook instead of treating this file as a reset
   procedure.
