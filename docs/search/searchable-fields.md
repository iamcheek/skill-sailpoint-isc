# Searchable Fields Reference

Source: https://documentation.sailpoint.com/saas/help/search/searchable-fields.html

Field names below are written exactly as they appear in a query — e.g. `attributes.location`, `@accounts(source.name)`. See [building-query.md](building-query.md) for query syntax mechanics (nested `@object(...)` syntax, wildcards, ranges, etc.).

Fields are grouped by "level":
- **First-level** — a scalar field with the actual searchable value (e.g. `name`).
- **Second-level** — an object field whose own first-level sub-fields are queried with dot notation (e.g. `manager.name`).
- **Nested-level** — an object that can occur multiple times per record (e.g. an identity can have many `accounts`); queried with `@object(field:term)` syntax.

---

## Identities (`identities` index)

### First-level fields

| Field | Type | Description |
|---|---|---|
| `name` | string | Identity name/alias |
| `displayName` | string | Unique display name |
| `firstName` | string | First name |
| `lastName` | string | Last name |
| `id` | string | Technical ID in ISC |
| `email` | string | Unique email address |
| `created` | date (ISO 8601) | Account aggregation date |
| `modified` | date | Most recent change date |
| `phone` | string | Alternate phone number |
| `status` | string | Identity status |
| `synced` | date | Last synchronization date |
| `isManager` | bool | Whether identity manages others |
| `employeeNumber` | string | Unique employee number |
| `processingState` | string | `ERROR` if aggregation failed |
| `appCount` | integer | Apps with access |
| `accountCount` | integer | Source accounts |
| `accessCount` | integer | Total access items |
| `entitlementCount` | integer | Entitlements held |
| `roleCount` | integer | Roles assigned |
| `accessProfileCount` | integer | Access profiles assigned |
| `tags` | string | Assigned tags |

### Second-level fields

| Field | Description |
|---|---|
| `manager.name`, `manager.displayName`, `manager.id` | Manager details |
| `source.id`, `source.name` | Authoritative source information |
| `identityProfile.name`, `identityProfile.id` | Profile details |
| `owns.sources.name`, `owns.sources.id` | Owned sources |
| `owns.accessProfiles.name`, `owns.accessProfiles.id` | Owned access profiles |
| `owns.entitlements.name`, `owns.entitlements.id` | Owned entitlements |
| `owns.roles.name`, `owns.roles.id` | Owned roles |
| `owns.governanceGroups.name`, `owns.governanceGroups.id` | Owned governance groups |
| `owns.apps.name`, `owns.apps.id` | Owned applications |
| `owns.fallbackApprover` (boolean) | Fallback reviewer status |
| `attributes.uid` (string) | UID from authoritative source |
| `attributes.firstname`, `attributes.lastname` | Name attributes |
| `attributes.cloudAuthoritativeSource` (string) | Source technical ID |
| `attributes.cloudLifecycleState` (string) | Lifecycle state |
| `attributes.identityState` (string) | Person state: `ACTIVE`, `INACTIVE_LONG_TERM`, or `INACTIVE_SHORT_TERM`. For currently active people, filter with `attributes.identityState:ACTIVE`; do not use top-level `status:ACTIVE` as a substitute, since that is ISC identity-record status. |
| `attributes.phone` (string) | Alternate phone |
| `attributes.displayName` (string) | Display name |
| `attributes.identificationNumber` (string) | Employee number |
| `attributes.workPhone` (string) | Work phone |
| `attributes.email` (string) | Email address |
| `attributes.startDate` (date) | Employment start date |
| `attributes.endDate` (date) | Employment end date |
| `attributes.nextProcessing` (date) | Next identity processing date |
| `attributes.personalEmail` (string) | Alternate email address |

### Nested-level fields (require `@object(...)` syntax)

**`@apps(...)`**

| Field | Type | Description |
|---|---|---|
| `id` | string | App technical ID |
| `name` | string | App display name |
| `source.name` | string | Source granting app |
| `source.id` | string | Source technical ID |
| `account.id` | string | Account technical ID |
| `account.accountId` | string | Account display name |

**`@accounts(...)`**

| Field | Type | Description |
|---|---|---|
| `id` | string | Account ID |
| `name` | string | Account display name |
| `accountId` | string | `aiqAccountName` or display name |
| `disabled` | bool | Account disabled status |
| `locked` | bool | Account locked status |
| `privileged` | bool | Privileged account marking |
| `manuallyCorrelated` | bool | Manual correlation status |
| `passwordLastSet` | date (ISO 8601) | Last password change |
| `created` | date | Account creation date |
| `entitlementAttributes.memberOf` | string | Entitlement attributes (field name varies by source) |
| `source.name` | string | Source display name |
| `source.id` | string | Source technical ID |
| `source.type` | string | Source type |

**`@access(...)`**

| Field | Type | Description |
|---|---|---|
| `id` | string | Access item technical ID |
| `type` | string | Access item type |
| `displayName` | string | Access item display name |
| `name` | string | Access item name |
| `description` | string | Access item description |
| `privileged` | bool | Privileged marking |
| `attribute` | string | Entitlement attribute type |
| `value` | string | Entitlement value |
| `disabled` | bool | Enabled/disabled status |
| `source.name` | string | Source display name |
| `source.id` | string | Source technical ID |
| `owner.id` | string | Owner technical ID |
| `owner.name` | string | Owner name |
| `owner.displayName` | string | Owner display name |

---

## Entitlements (`entitlements` index)

### First-level fields

| Field | Type | Description |
|---|---|---|
| `id` | string | Technical ID |
| `displayName` | string | Display name |
| `name` | string | Entitlement name |
| `description` | string | User-entered description |
| `privileged` | bool | Privileged status (`TRUE` = HIGH privilege level) |
| `requestable` | bool | Requestable status |
| `attribute` | string | Source attribute defining entitlement type |
| `value` | string | Source value for entitlement |
| `modified` | date | Last change date |
| `synced` | date | Last synchronization date |
| `tags` | string | Assigned tags |

### Second-level fields

| Field | Type | Description |
|---|---|---|
| `source.name`, `source.id` | string | Source name/ID |
| `segments.name`, `segments.id` | string | Segment name/ID |
| `owner.name`, `owner.id` | string | Owner identity name/ID |
| `privilegeLevel.direct` | string | Directly assigned privilege level (`HIGH`, `MEDIUM`, `LOW`, `NONE`) |
| `privilegeLevel.setBy` | string | User who set privilege level |
| `privilegeLevel.inherited` | string | Inherited privilege level |
| `privilegeLevel.effective` | string | Actual (resolved) privilege level |
| `privilegeLevel.setByType` | string | Assignment method: `OVERRIDE`, `CONNECTOR_CRITERIA`, `CUSTOM_CRITERIA`, `SINGLE_LEVEL_CRITERIA`, or `null` |

### Nested-level fields

**`@accessModelMetadata(...)`**

| Field | Type | Description |
|---|---|---|
| `key` | string | Attribute technical name |
| `name` | string | Attribute display name |
| `type` | string | Attribute type (`custom` or `governance`) |
| `description` | string | Attribute description |
| `value` | string | Value technical name |
| `valueName` | string | Value display name |

---

## Access Profiles (`accessprofiles` index)

### First-level fields

| Field | Type | Description |
|---|---|---|
| `created` | date (ISO 8601) | Creation date |
| `description` | string | User-entered description |
| `entitlementCount` | integer | Entitlements in profile |
| `id` | string | Technical ID |
| `modified` | date | Last change date |
| `name` | string | Profile name |
| `requestable` | bool | Requestable status (case sensitive) |
| `synced` | date | Last synchronization date |
| `tags` | string | Assigned tags |

### Second-level fields

| Field | Type | Description |
|---|---|---|
| `owner.name`, `owner.id` | string | Owner name/ID |
| `owner.type` | string | Owner entity type (always `identity`) |
| `owner.email` | string | Owner work email |
| `source.name`, `source.id` | string | Source name/ID |
| `segments.name`, `segments.id` | string | Segment name/ID |

### Nested-level fields

**`@entitlements(...)`**

| Field | Type | Description |
|---|---|---|
| `attribute` | string | Entitlement attribute type |
| `description` | string | Entitlement description |
| `id` | string | Entitlement technical ID |
| `name` | string | Entitlement name |
| `value` | string | Entitlement value |

**`@apps(...)`**

| Field | Type | Description |
|---|---|---|
| `id` | string | App technical ID |
| `name` | string | App name |
| `description` | string | App description |
| `owner.email` | string | Owner email |
| `owner.type` | string | Owner entity type (always `IDENTITY`) |
| `owner.id` | string | Owner technical ID |
| `owner.name` | string | Owner name |

---

## Roles (`roles` index)

### First-level fields

| Field | Type | Description |
|---|---|---|
| `name` | string | Role name |
| `id` | string | Technical ID |
| `description` | string | User-entered description |
| `created` | date (ISO 8601) | Creation date |
| `modified` | date | Last change date |
| `enabled` | bool | Enabled status (case sensitive) |
| `requestable` | bool | Requestable status (case sensitive) |
| `synced` | date | Last synchronization date |
| `accessProfileCount` | integer | Access profiles granted |
| `entitlementCount` | integer | Direct entitlements granted |
| `dimensional` | bool | Whether role has dimensions configured |
| `dimensionCount` | integer | Number of dimensions configured |
| `dimensionSchemaAttributeCount` | integer | Attributes used for dimensions |
| `tags` | string | Assigned tags |

### Second-level fields

| Field | Type | Description |
|---|---|---|
| `accessProfiles.id`, `accessProfiles.name` | array | Access profile technical IDs/names |
| `owner.id`, `owner.name` | string | Owner ID/name |
| `owner.type` | string | Owner entity type (always `identity`) |
| `owner.email` | string | Owner work email |
| `segments.name`, `segments.id` | string | Segment name/ID |

### Nested-level fields

**`@entitlements(...)`**

| Field | Type | Description |
|---|---|---|
| `attribute` | string | Entitlement attribute type |
| `schema` | string | Entitlement type |
| `description` | string | Entitlement description |
| `id` | string | Entitlement technical ID |
| `name` | string | Entitlement name |
| `value` | string | Entitlement value |
| `hasPermissions` | bool | Whether associated permissions exist |

**`@accessModelMetadata(...)`**

| Field | Type | Description |
|---|---|---|
| `key` | string | Attribute technical name |
| `name` | string | Attribute display name |
| `type` | string | Attribute type (`custom` or `governance`) |
| `description` | string | Attribute description |
| `value` | string | Value technical name |
| `valueName` | string | Value display name |

**`@dimensions(...)`**

| Field | Type | Description |
|---|---|---|
| `id` | string | Dimension technical ID |
| `name` | string | Dimension name |
| `description` | string | Dimension description |
| `accessProfiles.id`, `accessProfiles.name` | string | Access profile ID/name |
| `entitlements.id`, `entitlements.name` | string | Entitlement ID/name |
| `entitlements.description` | string | Entitlement description |
| `entitlements.attribute` | string | Entitlement attribute type |
| `entitlements.value` | string | Entitlement value |
| `entitlements.sourceSchemaObjectType` | string | Entitlement type |
| `entitlements.privileged` | bool | Privileged marking |
| `entitlements.hasPermissions` | bool | Whether associated permissions exist |

**`@dimensionSchemaAttributes(...)`**

| Field | Type | Description |
|---|---|---|
| `name` | string | Identity attribute technical name |
| `displayName` | string | Identity attribute display name |
| `derived` | bool | Whether derived from identity profile (always `true`) |

---

## Events (`events` index)

See [audit-events-searching.md](audit-events-searching.md) and [audit-events-reference.md](audit-events-reference.md) for the full audit-event workflow and the exhaustive event-name list.

### First-level fields

| Field | Type | Description |
|---|---|---|
| `created` | date (ISO 8601) | Event start date |
| `name` | string | User-friendly event translation |
| `operation` | string | Performed action |
| `status` | string | Event result |
| `technicalName` | string | Normalized event name (pattern: `objects_operation_status`) |
| `details` | string | Event description |
| `type` | string | Event classification/category |
| `id` | string | Technical event ID |
| `ipAddress` | string | Target system IP address |
| `objects` | string | Object the event affects |
| `action` | string | Legacy audit report event name |
| `trackingNumber` | string | Event group ID |

### Second-level fields

| Field | Type | Description |
|---|---|---|
| `actor.name` | string | Identity, source, or system generating the event |
| `target.name` | string | Event recipient (identity, source, or system) |
| `attributes.*` | varies | Event-specific attributes (varies by event type); may include `errors`, `provisioningResult`, `sourceName`, `_before`/`_after` |
| `attributes.clientMetadata` | string (JSON blob) | Caller-supplied metadata attached to `ACCESS_REQUEST`/approval events, e.g. `{"snowRITM":"RITM1234567","origin":"ServiceNow Service Catalog Integration"}`. Full-text/field-scoped searchable here (`attributes.clientMetadata:RITM1234567`) even though the same field is **not** filterable on `accountactivities` or `/access-request-administration/v1` (structured object there, rejected by `filters=`) and doesn't exist at all on the `accountactivities` index. See [docs/investigations/access-request-lookup.md](../investigations/access-request-lookup.md) for the full ticket-number-to-access-request resolution workflow. |

### Searchable status values

- **Event status values:** `approved`, `detected`, `escalated`, `failed`, `forwarded`, `ignored`, `passed`, `processed`, `rejected`, `started`, `terminated`

---

## Account Activity (`accountactivities` index)

### First-level fields

| Field | Type | Description |
|---|---|---|
| `action` | string | Action performed |
| `status` | string | Overall account activity status |
| `id` | string | Technical request ID |
| `trackingNumber` | string | Tracking/auditing number |
| `created` | date (ISO 8601) | Action creation date |
| `modified` | date | Last activity date |
| `sources` | list | Sources involved (case sensitive) |
| `stage` | string | Progress state from source |
| `errors` | list of strings | Source-provided error messages — a **flat array of plain strings**, not structured objects (confirmed against `search/schemas/accountactivitydocument.yaml`). A connector's `requestId` may appear as embedded text inside a message, e.g. `"...(requestId: <uuid>)"` — this is not a separate field, so `errors.requestId:"..."` is not a valid query; only full-text search against `errors` will match it. |
| `warnings` | list | Source-provided warnings |

### Second-level fields

| Field | Type | Description |
|---|---|---|
| `requester.id`, `requester.name` | string | Requesting user/system ID/display name |
| `recipient.id`, `recipient.name` | string | Modified user/system ID/display name |

### Searchable status values

- **`status`:** `complete`, `failure`, `incomplete`, `pending` (overall outcome)
- **`stage`:** `completed`, `executing`, `terminated` (progress state)
- **Account request status** (individual component result, see `accountRequests.result.status` below): `pending`, `committed`, `finished`, `unverifiable`, `failed`, `retry`, `manual task created`, `identity now task`

### Nested-level fields

**`@originalRequests(...)`**

| Field | Type | Description |
|---|---|---|
| `accountId` | string | Source account ID |
| `op` | string | Operation type: `create`, `modify`, `lock`, `unlock`, `enable`, `disable`, `delete` |
| `source.id`, `source.name`, `source.displayName`, `source.type` | string | Source details |
| `provisioningTarget.id`, `provisioningTarget.name`, `provisioningTarget.type` | string | Fulfilling source details |
| `attributeRequests.op` | string | Attribute operation: `add`, `set`, `remove` |
| `attributeRequests.name` | string | Attribute name |
| `attributeRequests.value` | string | New attribute value |

**`@accountRequests(...)`**

| Field | Type | Description |
|---|---|---|
| `accountId` | string | Native identity/account ID |
| `op` | string | Operation type |
| `source.id`, `source.name`, `source.displayName`, `source.type` | string | Source details |
| `provisioningTarget.id`, `provisioningTarget.name`, `provisioningTarget.type` | string | Fulfilling source details |
| `result.status` | string | Individual action status |
| `result.errors` | list of strings | Action-specific error messages — flat strings, same embedded-`requestId`-as-text caveat as the top-level `errors` field above (confirmed against `search/schemas/accountrequest.yaml`) |
| `result.warnings` | string | Action-specific warnings |
| `result.ticketId` | string | External system ticket ID |
| `attributeRequests.op` | string | Attribute operation |
| `attributeRequests.name` | string | Attribute name |
| `attributeRequests.value` | string | New attribute value |

**`@expansionItems(...)`**

| Field | Type | Description |
|---|---|---|
| `accountId` | string | Modified account unique identifier |
| `cause` | string | Reason: `Role`, `AttributeSync`, `AttributeAssignment`, `ProvisioningPolicy` |
| `name` | string | Reason details |
| `source.name`, `source.id`, `source.type` | string | Source details |
| `attributeRequest.op` | string | Attribute operation |
| `attributeRequest.name` | string | Attribute name |
| `attributeRequest.value` | string | New attribute value |

**`@approvals(...)`**

| Field | Type | Description |
|---|---|---|
| `workItemId` | string | Approval item technical ID |
| `created`, `modified` | date | Approval creation/last-modification date |
| `result` | string | Approval status: `Finished` or `Rejected` |
| `comments.commenter` | string | Commenter display name |
| `comments.comment` | string | Comment text |
| `comments.date` | date | Comment date |
| `owner.name`, `owner.id` | string | Reviewer name/ID |
