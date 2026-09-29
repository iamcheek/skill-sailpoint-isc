# Access Request Lookup ("What happened to this request?")

A "front door" playbook: someone hands you an external ticket number (ServiceNow RITM/REQ, or similar from another ITSM) and asks what happened to the access request behind it. Unlike [provisioning.md](provisioning.md), this doesn't start from a known identity/source/failure — it starts from an opaque ticket ID and nothing else. Load this playbook first for that shape of question; hand off to provisioning.md only if the trail leads to an actual failure.

## Why this needs its own path

Ticket numbers (e.g. `RITM1234567`) are not a native ISC field — they arrive embedded in `clientMetadata`, a caller-supplied object attached to an access request by whatever submitted it (ServiceNow Service Catalog integration, a custom UI, etc.). `clientMetadata` is **not filterable** via the `filters=` query syntax on `/access-request-status/v1` or `/access-request-administration/v1` — confirmed in live testing: `filters=clientMetadata co "RITM1234567"` and `filters=clientMetadata isnull` both return `400 Bad request content`. It's a structured object, not a queryable string/date/enum field.

## Step 1 — Resolve the ticket number to an `accessRequestId` via the `events` index

`clientMetadata` **is** indexed as searchable text on the `events` search index, under `attributes.clientMetadata` (a JSON-string blob) — but **not** on `accountactivities`. Verified in live testing:
- `_exists_:attributes.clientMetadata` on `accountactivities` → **0** records (field doesn't exist on that index at all)
- The same query on `events` → matches

So: search `events`, not `accountactivities`, for the ticket string.

```bash
sail --env <name> api post /search/v1 --body '{
  "indices": ["events"],
  "query": { "query": "attributes.clientMetadata:RITM1234567" },
  "limit": 250
}' --jsonpath "$" 2>&1 | tail -1
```

Field-scoping (`attributes.clientMetadata:<value>`) instead of a bare unscoped string search is the safer default — it guards against a coincidental match in some unrelated free-text field (e.g. a requester comment) inflating your result set. Verified in live testing: field-scoped and unscoped queries returned identical counts for sample ticket numbers, so scoping cost nothing in that test; don't assume that holds everywhere.

**Tokenization check (do this once per tenant, not per lookup):** confirm the analyzer treats the full ticket string as one token, not splitting the alphanumeric prefix from the digits. Verify with a substring probe one digit longer than a real ticket number; a zero-result response indicates no false-positive collision between adjacent ticket numbers. If your ticket-numbering scheme differs (e.g. shorter or differently zero-padded), re-verify before trusting result counts.

Each matching event carries `attributes.accessRequestId` (present on every event in the chain except the very first `AccessRequestRequested`/`ACCESS_REQUEST_STARTED` event, where it's `null` — use `trackingNumber` from that same event instead, which equals the `accessRequestId` once the request exists) and `attributes.clientMetadata` (which confirms the ticket number and origin system).

## Step 2 — Pull the request's approval/provisioning status

```bash
sail --env <name> api get /access-request-administration/v1 -H 'X-SailPoint-Experimental: true' \
  --query 'filters=accessRequestId eq "<accessRequestId-from-step-1>"' --jsonpath "$" 2>&1 | tail -1
```

`accessRequestId` **is** a documented, working filter field (unlike `clientMetadata`) — see [docs/api/endpoints.md](../api/endpoints.md). This returns `state` (e.g. `REQUEST_COMPLETED`), `approvalDetails[]` (who approved, when, any forwarding history), and `accessRequestPhases[]` (`APPROVAL_PHASE`/`PROVISIONING_PHASE` with `result`: `SUCCESSFUL`/etc.) — a starting point for "did this go through," but see Step 2.5 before treating it as decisive.

Use `/access-request-status/v1` instead (no experimental header needed) if the request belongs to the current authenticated user; use `/access-request-administration/v1` (ORG_ADMIN-scoped, requires `X-SailPoint-Experimental: true`) for looking up someone else's request — see [docs/api/endpoints.md](../api/endpoints.md) for the full split.

## Step 2.5 — Don't trust `state`/`completionStatus` alone to mean "it went through" (or "it didn't")

**Neither `/access-request-status/v1`'s top-level `state`/`approvalDetails[].status`, nor `/account-activities/v1/<id>`'s `completionStatus`/`items[].provisioningStatus`, reliably indicate whether the item was actually granted.** Confirmed in live testing, in both directions:
- A request showing `state: REJECTED` and `approvalStatus: EXPIRED` had actually succeeded — the entitlement was present on the account.
- A request showing `completionStatus: SUCCESS` and `provisioningStatus: FINISHED` — identical field values to a genuine grant — had actually been rejected by the approver, with no entitlement present.

`completionStatus: SUCCESS` appears to mean "the record's own processing pipeline completed without a technical/connector error," not "the requested item was provisioned." A cleanly-rejected request closes out its pipeline the same way a cleanly-granted one does.

The only reliable check is a live lookup: get the entitlement's `value` (`/entitlements/v1/<id>`), find the identity's account on that source (`/accounts/v1?filters=identityId eq "..." and sourceId eq "..."`), then check `/accounts/v1/<accountId>/entitlements` for that value. Do this before reporting a request's outcome as fact — don't stop at Step 2's fields even if they look decisive.

## Step 2.6 — Check `removeDate` before concluding "access is fine"

**`REQUEST_COMPLETED`/`SUCCESSFUL` only means the grant succeeded — it says nothing about whether access is still active right now.** The same response object from Step 2 carries top-level `startDate` and `removeDate` fields. If `removeDate` is non-null and in the past, the access was time-bound and has since been auto-revoked, even though the original request completed cleanly. This is a real, seen-in-practice cause of "why doesn't this person have the access anymore" that looks identical to a successful grant if you stop at `state`.

Both fields are on the same object Step 2 already returned — no extra call needed:

```bash
sail --env <name> api get /access-request-administration/v1 -H 'X-SailPoint-Experimental: true' \
  --query 'filters=accessRequestId eq "<accessRequestId>"' --jsonpath "$" 2>&1 | tail -1 \
  | jq '.[] | {state, startDate, removeDate}'
```

Gotchas confirmed in live testing:
- **`removeDate`/`startDate` are not filterable** via `filters=` on `/access-request-administration/v1` — `filters=removeDate isnull` and `filters=removeDate ne null` both return `400 Bad request content`. There's no way to server-side-filter "show me every time-bound request" on this endpoint; you have to pull records and check the field client-side.
- On the **`events`** index, the equivalent field is `attributes.currentlyAssignedRemoveDate` — but it's stored as an **epoch-millis string** (e.g. `"1785481200000"`), not ISO 8601. Don't compare it to a `YYYY-MM-DD` string; convert to/from epoch millis first.
- Once access is active, the scheduled removal date does **not** surface on the identity's current access list (`identities` index, `@access(...)` nested object has no `removeDate` field) — you can only see it by going back to the original access-request record via this playbook, not by inspecting the identity directly.
- A request that already auto-expired will still show `state: REQUEST_COMPLETED` — expiration doesn't change the request's own state field. The only tell is `removeDate` being in the past.

If `removeDate` explains the "loss" of access, that's a normal, expected lifecycle outcome (not a provisioning failure) — report it as such and stop; there's nothing to hand off to provisioning.md.

## Step 3 — Pull the provisioning plan and native identity (if you need account-level detail)

The same `accessRequestId` doubles as the account-activity ID — no separate lookup needed:

```bash
sail --env <name> api get "/account-activities/v1/<accessRequestId>" --pretty
```

This returns `items[]`, each with `nativeIdentity` (the target account's UUID on the source), `operation` (`ADD`/etc.), `provisioningStatus`, `sourceId`, and its own `clientMetadata` echo. Verified in live testing: this resolved cleanly for sample requests without any intermediate `trackingNumber` search step (contrast with provisioning.md's Source 1, which searches by identity name first — here you already have the exact ID).

## Step 4 — Hand off to provisioning.md only if something actually failed

If Step 2's `state`/`accessRequestPhases[].result` shows a failure (not `REQUEST_COMPLETED`/`SUCCESSFUL`), you now have everything provisioning.md's clarifying-questions table would otherwise have to ask for up front — `sourceId`, `nativeIdentity`, the UTC `created`/`modified` timestamps, and the identity involved (`targetIdentitySummary`). Skip straight to [provisioning.md Source 2 (Platform Logs)](provisioning.md#source-2--platform-logs-the-actual-error-detail) using those values; there's no need to re-derive them via provisioning.md's Source 1 search-by-identity-name step, since this playbook already got you further than that step would.

If the request completed successfully **and** Step 2.6 shows no past `removeDate`, there's nothing to hand off — report the status and stop. If Step 2.6 shows the access was time-bound and has since expired, that's also not a hand-off case — see Step 2.6 for why.

## Quick reference

| Question | Answer |
|---|---|
| Can I filter `/access-request-administration/v1` by `clientMetadata` directly? | No — 400 error, it's not a filterable field. |
| Where is the ticket number actually searchable? | `events` index, `attributes.clientMetadata` (full-text). Not `accountactivities` — that index doesn't carry the field at all. |
| How do I get from a ticket number to an `accessRequestId`? | Full-text/field-scoped search on `events`, then read `attributes.accessRequestId` (or `trackingNumber` on the first event in the chain). |
| How do I get status once I have the `accessRequestId`? | `filters=accessRequestId eq "..."` on `/access-request-status/v1` or `/access-request-administration/v1` — this filter is documented and works. |
| Why does someone no longer have access even though the request completed? | Check `removeDate` on that same record (Step 2.6) — a past `removeDate` means it auto-expired; `state` alone won't show this. |
| Does `state`/`completionStatus` reliably tell me if the item was actually granted? | No — confirmed both directions live (Step 2.5): a `REJECTED`/`EXPIRED` request had actually succeeded, and a `SUCCESS`/`FINISHED` request had actually been rejected. Always confirm with a live entitlement check on the account. |
| Can I filter by `removeDate` to find all time-bound requests? | No — 400 error, same as `clientMetadata`. Must pull records and check client-side. |
| How do I get the native account/identity touched? | `/account-activities/v1/<accessRequestId>` — same ID, no separate join needed. |
