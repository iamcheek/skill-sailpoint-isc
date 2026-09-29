# Provisioning Investigation Workflow

## Before you start: clarifying questions

Ask whatever's missing from this list before pulling logs — each one heads off a specific detour seen in past investigations. Skip any already answered by the user's message. This list is expected to grow/change as new detours turn up; add a row whenever a retro surfaces one.

| Ask | Why it matters | Skip if... |
|---|---|---|
| Which source/connector is this for (e.g. Salesforce SaaS, ServiceNow, or another connector)? | Without it you're grepping raw connector logs blind across every source in the tenant. Often inferable from a distinctive URI/hostname in the pasted error, but confirm rather than assume. | Named in the error text or the user's message |
| Which identity, account, or trackingNumber does this concern? | `account-activities` search (Source 1) needs an identity name or trackingNumber to be fast; without one you have to reverse-search from logs, which is slower and error-prone. | Already given |
| What's the actual goal — locate this one log entry, or understand *why* it failed? | "Find the entry" and "diagnose root cause" need different approaches (log grep vs. account-activity plan + accounts lookup). Defaulting to log-search when the real ask is root-cause wastes a round of searching. | Already stated, or obviously one or the other from phrasing |
| Roughly when did this happen, and in what timezone? | Needed to scope a log-pull window; "last hour" or "yesterday afternoon" must be converted to UTC before pulling ([timezone gotchas](#timezone--window-gotchas)). | A UTC or clearly-zoned timestamp is already given |
| Where did any ID in the error (trackingId, requestId, connector-instance ID) come from — do you know it's unique to this one event? | IDs pasted from error text can be reused/batch-level identifiers rather than unique correlation keys (seen in practice: one `trackingId` shared across 8+ unrelated errors). Don't assume uniqueness — verify cardinality (`grep -c`) before treating it as a reliable filter. | N/A — always worth a quick cardinality check once you have logs, even if you don't ask this aloud |

When asked to investigate provisioning failures or activity for an identity, **always combine these data sources** — Source 1 and Source 2 in parallel, then Source 3 once you know which item/account failed:

## Source 1 — Account Activities (the provisioning plan)

**Step A: Search for activities by identity**
```bash
sail api post /search/v1 --pretty --body '{"query":{"query":"recipient.name:\"<name>\""},"indices":["accountactivities"]}'
```
Extract `trackingNumber` and top-level `errors[]` (which contain `requestId`s) from each result.

**Step B: Fetch each activity by trackingNumber for the full provisioning plan**
```bash
sail api get /account-activities/v1/<trackingNumber> --pretty
```
This shows every individual `items[]` entry — attribute, operation, `provisioningStatus` (FINISHED/FAILED), sourceId, and value. The `errors[]` on the response contain `requestId`s for connector log correlation.

## Source 2 — Platform Logs (the actual error detail)

> **Scope:** `sail conn logs` only returns logs for **SaaS-based connectors**. Virtual appliance (VA)-based connectors write logs locally on the VA, not here.

### Step 0: Establish the UTC timestamp before pulling logs

Before pulling any logs, confirm the UTC timestamp of the request from the account activity (`created` field). Do not guess based on local time or "yesterday" — a request the user describes as happening "yesterday afternoon" may span two UTC calendar days depending on timezone offset. Always convert to UTC first.

**Option A — CLI (preferred for SaaS connectors, use this first):**
```bash
sail conn logs --start "<ISO8601-UTC>" --stop "<ISO8601-UTC>" 2>&1 > /tmp/conn_logs_<window>.txt
grep "<requestId>" /tmp/conn_logs_<window>.txt
```
Supports `--start`, `--stop`, `--target-id`, `--target-name`, `--level`, and `--request-id` flags. Use a tight window (±1-2 hours around the known UTC timestamp) rather than pulling a full day — it's faster and avoids large file sizes.

> **Verified in live testing:** `--request-id` and `--target-id` both filter server-side correctly — see [docs/cli/README.md](../cli/README.md#sail-conn-logs) for the comparison data. An older version of this note warned the flag "may be silently ignored" — that did not hold in this test, though re-verify against a same-window unfiltered baseline if you ever see it happen again.
>
> **Separate gotcha:** even with server-side filtering, the requestId you searched for may only appear nested inside `message.requestId` on the log lines with the actual error detail — the top-level `requestID` field on those lines can be the connector's own internal invocation id instead. Check both fields when correlating. See [docs/cli/README.md](../cli/README.md#sail-conn-logs) for an example.

**Option B — API directly (`/platform-logs/v1/query`):**
```bash
sail api post /platform-logs/v1/query -H 'X-SailPoint-Experimental: true' --body '{"filter":{"startTime":"<ISO8601-UTC>","endTime":"<ISO8601-UTC>"},"nextToken":""}' --pretty
```
This endpoint is experimental and **requires** the `X-SailPoint-Experimental: true` header — omitting it returns a 400. The body requires a `filter` object with at least `startTime`. Available filter fields: `startTime`, `endTime`, `component`, `logLevels`, `targetID`, `targetName`, `requestID`, `event`.

> **Verified in live testing:** `requestID`, `logLevels`, and `targetName` filters are all honored server-side — confirmed by comparing filtered vs. unfiltered results over the same time window. An earlier version of this doc warned that `requestID` was silently ignored; that does not hold on the tested API version. If you see all logs returned regardless of `requestID`, re-verify against a same-window unfiltered baseline before concluding the filter is broken again — it may be tenant- or version-specific.

> **Response shape:** `{"nextToken": "<string, empty when done>", "logs": [...]}`. Each log entry has: `tenantID`, `timestamp`, `level` (`INFO`/`ERROR`, uppercase), `event`, `component`, `targetID`, `targetName`, `sourceId`, `requestID`, `message`, `eventStatus`, `eventSummary`, `identity`. Paginate by re-posting the same filter with `nextToken` set to the previous response's `nextToken` until it comes back empty — note the endpoint may return several empty-`logs` pages with a non-empty `nextToken` before truly finishing when a filter narrows results heavily; keep paginating until `nextToken` is empty, not until `logs` is empty.

> **Note:** `/beta/platform-logs` (without `/query`) returns 200 with an empty body — it is not the correct endpoint.

> **Pagination requires a loop — script it up front, don't call page-by-page by hand.** A narrowed `requestID` filter commonly needs teens of pages before the real hits appear (verified in a narrowed-result investigation), so budget for a scripted loop, not a handful of manual calls. Use `--jsonpath "$.nextToken"` and `--jsonpath "$.logs"` (see [docs/cli/README.md](../cli/README.md#output-capture-pattern)) to get clean, directly-parseable values on stdout instead of regex-scraping the mixed stdout/stderr `--pretty` output for the JSON body.

#### Platform-logs `requestId` scope

> **`requestID` is a platform-log-only correlation key — it is not a searchable/filterable field anywhere else.** It does not exist as a field on `accountactivities`, `identities`, or any other `/search/v1` index (a query like `errors.requestId:"<id>"` returns `[]` — silently empty, not an error). To pivot from a log's `requestID` back to the provisioning-plan view, use the matching **`trackingNumber`** via `/account-activities/v1/<trackingNumber>` (Source 1 above) — that is the documented, reliable join key. As a fallback, a plain full-text search (no field prefix) for the raw ID string against `accountactivities` can surface the record, because SailPoint sometimes embeds the requestId as plain text inside the `errors[]` message (e.g. `"... (requestId: <id>)"`) — but that's a lucky text match on an unstructured string, not a queryable field, so don't rely on it as a general pattern.
>
> This `requestID` is also unrelated to the ISC Workflows feature's own, separately-scoped `requestId` (a different field entirely — the name just collides). For the log-entry-structure gotcha — a target `requestID` can be nested inside `message.requestId` rather than at the top level — see [docs/cli/README.md](../cli/README.md#sail-conn-logs).

> **One requestID can span multiple sources/targets.** A single automated account activity fans out `Disable`/`Update`/etc. commands to every connected source in the plan (e.g. Okta, ServiceNow, and multiple environments) concurrently. Platform-log entries for other targets can appear correlated in the same time window/connector process even though only one leg's response actually carries that literal `requestID` — don't assume every log line sharing a window belongs to the same target's outcome; check `sourceId`/`targetName` per entry.
>
> **Corollary — a shared requestID can also span multiple *distinct error types*, not just multiple targets.** Do not generalize from the first error found under a shared ID to every item using that ID. **Before attributing a cause to a group of failures that share a requestID/trackingId, trace each item individually** (its own `nativeIdentity`/native account → its own `invocationId` → its own attribute-level error text in the connector log).
>
> Treat "trace each item individually" above as a required step before stating a root cause, not an optional deeper dive to reach for only if something looks inconsistent.

### Timezone / window gotchas

- ISC API timestamps (`account-activities`, `errors[]`) are **UTC**
- Always confirm the UTC `created` timestamp from the account activity before choosing a log window — never infer from local time or relative terms like "yesterday"
- If the user's local time is UTC-5, a request they saw at "2:30pm" happened at **19:30 UTC** — pull the window in UTC accordingly
- When a requestId isn't found, the window is usually too narrow — widen it by ±30 min and re-pull rather than assuming the log is missing
- Save each pull to a distinct `/tmp/` file (e.g. `_logs_1900.txt`, `_logs_2030.txt`) so you can grep across cached windows without re-pulling

### Interpreting errors — stay within the evidence

When analyzing connector errors, only assert what the logs directly show. Do not infer the HTTP method or operation type from the error message alone — a "user already exists" 409 does not prove a POST/create was made. The stack trace (`commandType`, method names in the call stack) and debug-level `Executing endpoint:` / `SCIM Payload:` entries are the authoritative source. Enable debug logging on the connector and re-pull if the operation details are not visible at INFO/ERROR level.

> **`std:account:create` operations have no identity/email correlation at INFO/ERROR level.** Unlike `std:account:update` (whose "Running before account" log line names the target `nativeIdentity` directly), Salesforce SaaS create-account log lines carry only an opaque `invocationId` — verified in live testing: searching connector logs for a failing identity's email/username during a create-account investigation returned zero hits, because the identity's name/email never appears at that log level for `std:account:create`. **For a create-operation's actual provisioning plan (which attributes it tried to set), pull the account-activity instead — there is no `beforeCreate` hook that echoes the plan to the connector log the way there is for updates.** `/account-activities/v1/<trackingNumber>` remains reliable for this even when connector logs can't be tied to the identity. If you need to map a specific failing identity to its specific connector-log `invocationId` for a create op, debug-level logging is required — timestamp proximity between an activity's `created`/`completed` window and a nearby `invocationId`'s start/error time is not sufficient proof of identity, especially when multiple creates run concurrently in the same batch.

## Source 3 — Accounts endpoint (has the failure actually left the target account in a bad state?)

After identifying the failing item's `sourceId` and `nativeIdentity`/account UUID from Source 1, check whether the account's cached state in ISC actually reflects the failure, rather than assuming the alert text alone is current:

```bash
sail api get /accounts/v1 --query 'filters=nativeIdentity eq "<native-identity-or-uuid>"' --pretty
```

`nativeIdentity` is a valid filter field here (verified in live testing) even though it isn't in the filter-examples table in [filtering-sorting-pagination.md](../api/filtering-sorting-pagination.md) — use it when you have the account UUID from a connector log or account-activity item and want to pivot straight to the account record without going through `identityId`. Check the returned `disabled` field against what the operation was trying to set — e.g. for a failed `DISABLE`, `disabled: false` confirms the target account is still genuinely in the pre-failure state, not just that an old alert hasn't been updated.

## Why all three are needed

| Data source | Tells you |
|---|---|
| `account-activities` | What ISC tried to do, which items passed/failed (the plan view) |
| `connector logs` | *Why* it failed — actual target system API errors, codes, and messages |
| `accounts` (`/accounts/v1`) | Whether the failure actually left the account in a bad state right now, per ISC's own cached view |

## Root cause pattern: multiple accounts on one source breaks provisioning

**SailPoint ISC access requests operate at the identity level, not the account level.** When an access item (entitlement, role, permission set) needs to be granted, ISC resolves the item to a source, then looks for the identity's account on that source:
- No account exists on that source → ISC creates one
- Exactly one account exists → ISC modifies it
- **More than one account exists on that source → ambiguous target.** Unless the source has explicit multi-account correlation rules configured, there is no defined logic for ISC to pick which account to operate on, and provisioning fails.

**Accepted practice:** an identity should have exactly one account per source. ISC does support multi-account correlation, but it's a niche/opt-in configuration — standard provisioning plan construction assumes a single account per identity per source.

**Diagnostic fingerprint:** this failure mode can surface as a connector-instance schema-validation error shaped like:
```
Schema validation error in path: ... Required property 'identity' is missing ... Required property 'compound' is missing ... Value is null but should be string
```
on the `object_key`/`object_input` schema — the provisioning command was built without a single resolvable native account to target, so required identifier fields came through empty/null. When you see this error shape, jump straight to `/accounts/v1?filters=identityId eq "<id>"` filtered to the failing source, before spending time on connector logs — connector logs will show nothing for the exact failure timestamp either way, because the malformed request never reaches the connector's execution code (it's rejected by ISC's own orchestration layer, at the `connector-instances/<id>/commands` call, before invoking the connector). If the identity has 2+ accounts on that source, that's very likely the root cause.

## Remediation: manually reprocessing an identity after a resolved/transient failure

Once root cause is confirmed as something that self-resolves (for example, a transient connector-side lock) rather than a config problem needing a fix first, the next question is usually **"do you want me to manually reprocess this identity?"** or, when a batch of failures shares one root cause, **"I need you to manually reprocess a handful of identities"** — offer this proactively once you've verified the underlying condition has cleared, rather than waiting to be asked. Same endpoint and verification approach for both; a batch is just more `identityIds` in one call (up to 250) and more per-identity checks in step 2 below, not a different flow.

**Endpoint:** `POST /identities/v1/process` (requires `X-SailPoint-Experimental: true` header, `ORG_ADMIN`/`HELPDESK`). Body: `{"identityIds": ["<id>", ...]}`, 1–250 identityIds per call. Per its own spec description this endpoint's job explicitly includes step 3, "Enforce provisioning for any assigned accesses that haven't been fulfilled (e.g. failure due to source health)" — this is the correct tool for retrying a failed grant, not `synchronize-attributes` (that endpoint only recalculates attributes; it does not re-attempt failed role/entitlement provisioning).

**Getting the `identityId`:** the failed event's `target.name` (from Source 1/2 above) is an internal numeric ID, not the `identityId` GUID this endpoint needs. Resolve it via the account-activity: `GET /account-activities/v1/<trackingNumber>` → `targetIdentitySummary.id`.

> **Gotcha (verified in live testing): the task ID this endpoint returns is not trackable via `/task-status/v1/{id}`.** The `202` response is a `type: TASK_RESULT` DTO, but its `id` comes back as a dashed UUID (e.g. `<dashed-uuid>`) — a different format from the 32-char hex IDs `/task-status/v1` actually uses (confirmed against the task IDs returned by the endpoint). Looking the UUID up via `GET /task-status/v1/{id}` returns 404, and filtering `/task-status/v1?filters=id eq "<uuid>"` returns `[]`. The `TASK_RESULT` DTO shape is shared cosmetically across several distinct SailPoint job-tracking mechanisms (e.g. `identitysyncjob.yaml`'s job IDs use the same dashed-UUID format) — it does not imply they all resolve through the same status endpoint. No GET path in the current `identities` or `task-management` API spec accepts this UUID format. Don't spend time polling task-status for this job type — go straight to event-based verification instead.

**Verification — search the `events` index, don't poll a task status:**
1. Confirm the reprocessing pass itself ran: `action:IDENTITY_PROCESSING` events carry `technicalName: IDENTITY_PROCESSING_MANUAL_STARTED`/`IDENTITY_PROCESSING_MANUAL_PASSED`, with `target.name` matching the identity's internal numeric ID (not the identityId GUID) and `attributes.operation: MANUAL`.
2. Confirm the specific provisioning item actually succeeded this time (the identity-processing pass "passing" doesn't by itself prove the target grant went through) — look for a **new** `trackingNumber` after your call's timestamp with `ACCOUNT_MODIFY_PASSED`/`ENTITLEMENT_ADD_PASSED` for the same source/attribute/value that failed originally. Example query: `sail search query 'action:AddEntitlement AND created:[now-10m TO now]' --indices events`, then filter results to the target identity and attribute.
3. Only report success once both (1) and (2) confirm — a `PASSED` identity-processing event with `identitiesProcessed: 0` is not itself proof the failed item was retried; it's proof the pass ran.

**Batch reprocessing (multiple identities in one call):** don't assume one success implies the rest — a shared root cause (e.g. everyone hit the same lock) doesn't guarantee everyone clears at the same time once retried, especially if the retry itself creates new contention. Run step 2's per-identity check against **every** identityId in the batch, not just a sample, before reporting the batch as resolved. Consider testing one identity first (as a canary) before firing the full batch — use the first retry as a canary to confirm the endpoint/verification approach works.

## Connector-specific troubleshooting

Once you know which connector/source is involved and have an actual error message or symptom in hand, check [docs/connectors/](../connectors/README.md) for a per-connector reference file before root-causing from scratch — each file is a pattern-match table of known error signatures (vendor-documented and/or verified in live testing) that may already cover what you're seeing. This applies whether the investigation is provisioning-related or something else entirely (connection health, auth setup, aggregation, etc.).

Currently cataloged:
- [Salesforce SaaS](../connectors/salesforce.md)

Absence of a connector in that catalog doesn't mean the Source 1-3 methodology above doesn't apply — it means no one's built that connector's reference file yet.
