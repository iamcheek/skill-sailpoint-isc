# Web Services SaaS Connector

Generic connector reference for sources using SailPoint's **Web Services SaaS** connector. See
[README.md](README.md) for what belongs in these files; application-specific behavior belongs in the
integration's profile.

This file covers general Web Services SaaS configuration patterns: vendor capability boundaries,
schema relationships, response mapping, parent/child operations, pagination, and aggregation
verification. These still need to be checked against the source's current configuration and
connector version; re-read the source before relying on an operation's enabled state, pagination,
or mappings.

## Vendor documentation and capability boundaries

Reviewed 2026-11-11 against SailPoint's [Web Services SaaS documentation index](https://documentation.sailpoint.com/connectors/web_services/saas/help/llms.txt). This reference applies to **Web Services SaaS**, not automatically to a VA-based Web Services connector. A documented feature is not proof that a particular source, target API, connector version or license supports or enables it.

| Topic | Vendor references |
|---|---|
| Scope, setup and authentication | [Overview](https://documentation.sailpoint.com/connectors/web_services/saas/help/index.md), [connection settings](https://documentation.sailpoint.com/connectors/web_services/saas/help/connecting.md), [configuration and chaining](https://documentation.sailpoint.com/connectors/web_services/saas/help/configuring.md), [Connectivity Agent setup](https://documentation.sailpoint.com/connectors/web_services/saas/help/connectivity_agent_setup.md) |
| Operations and mappings | [HTTP operations](https://documentation.sailpoint.com/connectors/web_services/saas/help/http_operations.md), [keywords](https://documentation.sailpoint.com/connectors/web_services/saas/help/keywords.md), [JSON/XML examples](https://documentation.sailpoint.com/connectors/web_services/saas/help/operations_reference.md), [entitlements](https://documentation.sailpoint.com/connectors/web_services/saas/help/entitlements.md) |
| Aggregation and extensions | [Pagination](https://documentation.sailpoint.com/connectors/web_services/saas/help/pagination.md), [partitioning](https://documentation.sailpoint.com/connectors/web_services/saas/help/partitioning.md), [datasets](https://documentation.sailpoint.com/connectors/web_services/saas/help/datasets.md), [customizers](https://documentation.sailpoint.com/connectors/web_services/saas/help/customizers.md) |
| Settings and diagnostics | [Source settings](https://documentation.sailpoint.com/connectors/web_services/saas/help/source_settings.md), [configuration parameters](https://documentation.sailpoint.com/connectors/web_services/saas/help/configuration_parameters.md), [troubleshooting](https://documentation.sailpoint.com/connectors/web_services/saas/help/troubleshooting/index.md), [timeouts](https://documentation.sailpoint.com/connectors/web_services/saas/help/troubleshooting/timeout_errors.md) |

- The [overview](https://documentation.sailpoint.com/connectors/web_services/saas/help/index.md) documents REST, SOAP and GraphQL with JSON/XML responses and multiple account, entitlement and provisioning operations. The [connection guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/connecting.md) says provisioning is disabled by default; check the actual source and each target endpoint before claiming writes are enabled or supported.
- **Authentication:** An API Token is supplied as an access token; OAuth 2.0 is a separate mode with supported grant types. The connection guide says built-in OAuth token requests use `application/x-www-form-urlencoded`; use the vendor's Custom Authentication guidance when the target requires a different token request. Do not hard-code or expose credentials, tokens or authorization headers in URLs, examples, logs or source captures; use approved secret storage and sanitized projections. A placeholder alone is not evidence that a secret is safely protected.
- **Connectivity Agent:** SailPoint's [setup guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/connectivity_agent_setup.md) says it supports REST only, not custom authentication; enabling it accepts AI terms, and switching to Standard Setup is permanent. Treat its suggestions as recommendations to verify against target documentation and actual configuration, not tested requirements. Do not enable it or switch modes without authorization.
- **Datasets:** The [datasets guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/datasets.md) identifies Web Services as a generic source where users create datasets/resources, configure Resource Aggregation and set owner correlation. Its generic dataset section also says datasets cannot be created or deleted, so confirm the Web Services-specific behavior and feature availability before relying on it. A dataset resource is not automatically an account entitlement or proof of aggregation.

## Request construction, outcomes and safety

- [HTTP operations](https://documentation.sailpoint.com/connectors/web_services/saas/help/http_operations.md) distinguish an operation's **name** from its **type**, method, URL, request/response mappings and success codes. GraphQL has its own request type; the guide limits placeholder/Velocity support to GraphQL variables. Verify actual target contracts, response schema and configuration rather than deriving a request from an example.
- **Missing placeholders can change a request silently:** SailPoint documents that an unresolved placeholder can remove a JSON body key/value, remove a header key/value, or remove the value from a URL. Inspect sanitized rendered requests for missing required fields and scope filters; an HTTP 200 alone is not proof the intended request was sent. The [keyword reference](https://documentation.sailpoint.com/connectors/web_services/saas/help/keywords.md) distinguishes `$plan.*$`, `$response.*$`, `$application.*$`, `$getobject.*$` and `$authenticate.*$`; verify each placeholder's scope and casing in the actual operation.
- **Success and retries:** The documented default success-code range is `2xx`; a response body can still contain an error. [Source settings](https://documentation.sailpoint.com/connectors/web_services/saas/help/source_settings.md) say `possibleHttpErrors` can recognize target-specific codes/messages and support OAuth token-refresh retry, but is **not applicable to child endpoints**. The [parameter guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/configuration_parameters.md) documents automatic retries for 429/503, up to three per request: positive `retryDelay` takes precedence, then numeric `Retry-After`, then 30 seconds. Verify current version/config and the target's idempotency and partial-write semantics before retrying provisioning.
- **Provisioning shape:** The parameter guide describes `createAccountWithEntReq`, `addRemoveEntInSingleReq`, `updateAttrWithChangePassword` and `skipGetObjectInCreate`; they can change request grouping or post-create readback. Do not assume defaults or enable these flags without checking the source, target contract, approval and recovery plan. [Troubleshooting](https://documentation.sailpoint.com/connectors/web_services/saas/help/troubleshooting/index.md) documents `handleNullResponseInDisable` for a specific null-response/missing-output symptom; verify the actual response before proposing a change.
- **Status and delta:** [Source settings](https://documentation.sailpoint.com/connectors/web_services/saas/help/source_settings.md) say the Account Enable Status Attribute marks matching values enabled and all other accounts disabled; the lock-status setting can exclude accounts from provisioning. Delta aggregation and deletion markers depend on target support. Verify the target's complete status vocabulary, source configuration and population impact before proposing changes.
- **Timeouts:** [Timeout guidance](https://documentation.sailpoint.com/connectors/web_services/saas/help/troubleshooting/timeout_errors.md) distinguishes aggregation, health check, provisioning, pass-through and API timeouts. Diagnose the actual limit and affected endpoint before proposing changes; do not automatically raise timeouts or run test connections/aggregations as part of read-only discovery.

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
3. Configure the group aggregation **operation type** for that group schema's object type. The
   [vendor entitlement guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/entitlements.md)
   documents group-specific aggregation, Get Object and add/remove operations, but its displayed
   labels and serialized examples use inconsistent spacing. Inspect the actual source's operation
   type and schema association; do not confuse it with the user-defined operation name or copy a
   vendor PATCH example as an authoritative payload.

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
persisted or associated to an account; verify the connector scan, persistence and account
association separately, as described in
[source aggregation verification](../../investigations/source-aggregation.md#verification-guidance).
For Web Services SaaS specifically, an HTTP 200 response or a log line such as `Entitlements: N` proves connector output, not
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
handling is explicitly configured. Check a sanitized or synthetic response and inspect the resulting
resource object; live preview, aggregation and source changes require authorization.

## Parent and child Web Services operations

Use a parent operation when its response contains the account fields needed for normal aggregation,
and use a child operation when a second endpoint is needed to enrich each parent resource. The
[configuration guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/configuring.md)
describes **Parent Endpoint** for chained operations and independent endpoints of the same operation
type. It also says that on existing sources the displayed Parent Endpoint changes have no effect and
legacy first-endpoint/child chaining applies; the guide does not define the source-version boundary.
Do not infer effective chaining from the UI alone: verify the source/version and actual behavior with
safe evidence or SailPoint. Connector operation variables can reference the parent response, for example:

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

Do not change it to an object such as `{ "ids": ["$response.id$"] }` unless that endpoint
actually documents the object form. A child operation can then map only the fields that are absent
from the parent response, reducing the chance that it replaces or drops parent attributes.

## General pagination and partitioning

The [pagination guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/pagination.md)
describes limit/offset, request-body, response marker/header and last-record paging. Steps are a
connector-specific language, one per line; compute arithmetic outside `TERMINATE_IF`. Use the target
API's documented ordering, cursor/offset rules and termination contract, then verify coverage and
unique identities rather than assuming a page count proves completeness.

> **Vendor documentation conflict:** The pagination guide says paging is supported only for the
> first account or entitlement aggregation operation instance, while the
> [configuration guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/configuring.md#configuring-multiple-independent-endpoints-for-aggregation)
> says each independent endpoint can have its own paging. The applicable version/conditions are not
> stated. Treat multi-endpoint paging as **unresolved** until verified for the source/version or with
> SailPoint; do not silently choose one rule.

The [partitioning guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/partitioning.md)
describes static and dynamic partitions: both require a Partitioned Account Aggregation operation;
dynamic partitioning also requires Get Partitions. Static and dynamic partition lists are concatenated,
so assess overlap, omissions, ordering, target rate limits and duplicate identities. Normal and
partitioned account aggregation use separate operations; don't assume partitioning or delta aggregation
is configured just because the connector supports it. Source changes and live tests require approval.

## Source-specific pagination checks

Treat pagination as part of the source-specific API contract. Before enabling or changing an
operation, verify:

- whether page numbering is zero- or one-based;
- whether the API uses page/offset, a continuation token, or another mechanism;
- whether the response reports the actual page size and total;
- whether the ordering is deterministic across pages; and
- whether the connector's pagination script uses the response fields and syntax supported by the
  current connector version.

Prefer a documented deterministic ordering when the API supports one. An unstable order between
offset pages can cause duplicates or omissions; changing page size alone is not a reliable fix.
Compare the number of scanned records with the number of unique native identities after aggregation.
If the API returns `page`, `pageSize`, and `total`, the pagination script can follow the server's
actual page size rather than assuming the requested one.

## Endpoint-level customizers

Web Services SaaS supports both standard connector hooks and endpoint-specific named customizations.
The [vendor customizer guide](https://documentation.sailpoint.com/connectors/web_services/saas/help/customizers.md)
describes endpoint `isBeforeCustomizer`/`isAfterCustomizer` flags and named handlers such as
`customizedOperation('<endpoint-unique-name>:before', handler)` or
`customizedOperation('<endpoint-unique-name>:after', handler)`.
The source endpoint flag, exact operation name, linked customizer and deployed handler must all be
verified for the source/version; a standard `beforeStdAccountList`/`afterStdAccountList` hook is not
the same mechanism. Before/after handlers may see request configuration or raw/processed response
objects: preserve required fields and avoid logging headers, credentials or raw personal data. Do not
copy vendor examples that hard-code key material, log full requests or suppress errors. Linking,
uploading, changing flags or live testing needs explicit authorization; see the shared
[customizer reference](../development/customizers.md).

## Aggregation ordering and troubleshooting

When an account schema references a group schema, make the group operation name, schema reference,
and aggregation order explicit. At minimum, verify:

- the group schema ID referenced by the account attribute is the current one;
- the account attribute's multiplicity matches the target relationship and it has both entitlement/group flags;
- the group operation type matches the intended schema object type and its actual serialized source configuration;
- the group aggregation and account aggregation are enabled and run in the intended order;
- the response root and mappings produce resource objects with stable identities; and
- the final account resource still contains the entitlement attribute after all parent/child and
  after-operation processing.

A successful HTTP response does not rule out a later mapping or post-processing problem. When
specifically authorized, a controlled single-user aggregation can compare the parent response/resource,
child resource, final post-customizer User RO and persisted account. Protect sensitive data and compare
each stage rather than relying only on connector counts.

If a newly mapped account attribute is absent from the final User RO or persisted account even
though the parent/child HTTP calls returned it, compare the raw connector resource, the
post-customizer User RO, and the saved account. A customizer or after-operation step may be removing
or replacing the value; establish where the value disappears before changing the connector mappings.
If disabling the customizer for a single-user aggregation would help isolate it, do so only when
that is safe and approved.

Likewise, the connector's scanned-resource count is not necessarily the persisted unique-account
count; compare native identities after aggregation when diagnosing duplicates or dropped objects.

## How to use this during an investigation

1. Identify the source as **Web Services SaaS** and load this file for planning, discovery,
   implementation or troubleshooting; distinguish vendor guidance from source-specific observations.
2. Re-read the current source configuration before changing anything. UI edits can remove pagination
   steps, query parameters, mappings, or child operations, or change an operation's enabled state.
3. Separate HTTP output, connector resource objects, persisted entitlements, account associations,
   and post-customizer account resources.
4. Label observations as either general connector configuration guidance or verified behavior of a
   specific source/API. Do not promote a tenant-specific finding to a universal connector rule
   without independent verification.
5. If the symptom is a source reset or pending-task blocker rather than Web Services mapping or
   aggregation, use the applicable investigation playbook instead of treating this file as a reset
   procedure.
