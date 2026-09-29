# Response Fields by Resource

Key fields to look for in API responses, plus enum values for fields that only accept a fixed set of strings. For what endpoint returns each resource, see [endpoints.md](endpoints.md).

| Resource | Key fields |
|---|---|
| AccountActivity | `id`, `name`, `type`, `created`, `modified`, `completed`, `completionStatus`, `executionStatus`, `requesterIdentitySummary`, `targetIdentitySummary`, `items[]`, `errors`, `warnings` |
| AccountActivity `targetIdentitySummary.id` | The identity's GUID `identityId` (e.g. `9c0bc99fd28f46b289434fc8fd79973a`) — use this when you need the real `identityId` for another call (e.g. `/identities/v1/process`) and only have a `trackingNumber`/internal numeric ID from an `events` search (`target.name` there is the internal numeric ID, not this GUID). |
| AccountActivity `type` values | `CloudAutomated`, `IdentityAttributeUpdate`, `appRequest`, `LifecycleStateChange`, `AccountStateUpdate`, `AccountAttributeUpdate`, `CloudPasswordRequest`, `Certification`, `Identity Refresh`, `Lifecycle Change Refresh` |
| Identity | `id`, `name`, `alias`, `emailAddress`, `created`, `modified`, `processingState`, `identityStatus`, `isManager`, `managerRef`, `attributes` |
| Identity `identityStatus` values | `UNREGISTERED`, `REGISTERED`, `PENDING`, `WARNING`, `DISABLED`, `ACTIVE`, `DEACTIVATED`, `TERMINATED`, `ERROR`, `LOCKED` |
| Account | `id`, `sourceId`, `sourceName`, `identityId`, `nativeIdentity`, `uuid`, `disabled`, `locked`, `manuallyCorrelated`, `hasEntitlements`, `uncorrelated`, `authoritative`, `isMachine`, `attributes` |
| AccessProfile | `id`, `name`, `description`, `created`, `modified`, `owner`, `source`, `entitlements[]`, `requestable` |
| Role | `id`, `name`, `description`, `created`, `modified`, `owner`, `accessProfiles[]`, `requestable`, `dimensional` |
| AccessRequestStatus (`/access-request-status/v1`, `/access-request-administration/v1`) | `accessRequestId`, `state`, `requestType`, `requester`, `requestedFor`, `approvalDetails[]`, `accessRequestPhases[]`, `clientMetadata`, `startDate`, `removeDate`, `accountActivityItemId` |
| AccessRequestStatus `startDate`/`removeDate` | Time-bound access window. Both `null` for standing (non-expiring) grants. A non-null `removeDate` in the past means access has auto-expired even though `state` still reads `REQUEST_COMPLETED` — **not filterable** via `filters=` (confirmed 400 on both endpoints), so finding time-bound requests requires pulling records and checking client-side. See [docs/investigations/access-request-lookup.md](../investigations/access-request-lookup.md#step-25--check-removedate-before-concluding-access-is-fine). |

For presenting results to the user (formatting, summarizing large responses, highlighting requested fields), see the parent [SKILL.md](../../SKILL.md) Step 5.
