# Source Reset Runbook

Use this playbook when an ISC source needs its account data, entitlement data, or both reset. This is an **ISC-side data reset**, not a source disconnect/delete and not an aggregation. It does not send account deletion/provisioning changes to the connected application.

> **Rare, destructive operation — sandbox/new-source use is the norm.** Source reset is not a routine production operation. It is typically used while building or validating a new source in a sandbox. Treat production resets as exceptional; do not run one as ordinary troubleshooting or maintenance.
>
> **Explicit user confirmation is mandatory before any reset request.** Before sending a reset POST, explain the exact environment, source name/ID, and reset scope—accounts only, entitlements only, or full source (both)—including the data that will be removed. Obtain a clear confirmation from the user for that specific scope. Do not infer consent from a request to investigate, write this runbook, or run an aggregation. If confirmation is absent or ambiguous, stop without calling the endpoint. For production, also require the applicable change/operational approval; user confirmation alone does not replace it.

The procedure is based on the community VS Code extension's implementation of **Reset source**, **Reset accounts**, and **Reset entitlements**, and the `sailpoint-api-client` version pinned by that extension when reviewed. Confirm the current endpoint contract before relying on it if the extension or API client has changed.

Set the explicitly selected environment and target source ID before using the commands below:

```bash
ISC_ENV='<selected-environment>'
SOURCE_ID='<source-id>'
EMPTY_BODY_FILE='/tmp/isc-empty-reset-body'
: > "$EMPTY_BODY_FILE"
```

The reset endpoints take no request body, but `sail api post` requires `--body` or `--body-file`. Pass this zero-byte file; do not substitute a JSON object.

## Impact and safety gates

These calls are destructive to ISC's imported data for the selected source:

- Reset accounts removes all accounts for the source from ISC. A later account aggregation can recreate them.
- Reset entitlements removes all entitlement objects for the source from ISC.
- Reset source performs both in sequence: accounts first, then entitlements only if account reset completes with `Success`.

Before sending either POST, complete these steps in order; do not call a reset endpoint until step 2 is explicitly confirmed:

1. Select the intended ISC environment explicitly with the `sail` skill's environment workflow; never rely on the persisted default.
2. Verify the source ID and name in that environment (for example, `GET /sources/v1/{sourceId}`). Tell the user the exact environment, source name/ID, and data the reset will remove, then obtain their explicit confirmation to proceed.
3. Check for active/conflicting aggregation or reset tasks and avoid running resets concurrently with aggregation/provisioning activity. The task list supports filtering by `sourceId` and null `completionStatus`:
   ```bash
   sail --env "$ISC_ENV" api get /task-status/v1 --query "filters=sourceId eq \"$SOURCE_ID\" and completionStatus isnull" --pretty
   ```
4. Record any before-state counts or evidence needed for recovery. A reset itself does not reaggregate the source. If a task appears to be blocking the reset, diagnose it using [stale-task-remediation.md](stale-task-remediation.md); do not force-complete tasks as part of the reset procedure.
5. Ensure the API principal has permission to reset the selected source. The VS Code extension also asks for confirmation (or invokes its tenant-read-only guard); raw API calls do not provide that interactive safety check.

Do not run these POSTs merely to troubleshoot a missing entitlement association without understanding that all source accounts and/or all source entitlements will be removed from ISC.

## Underlying API endpoints

The extension's bulk reset calls use the source ID in the path and no JSON request body. The single-account remove endpoint is an additional API operation, not part of the extension's `Reset source` sequence. For `sail`, use the zero-byte body file described above because the CLI requires a body-file flag for POST:

| Operation | Method and endpoint | Result |
|---|---|---|
| Reset accounts | `POST /sources/v1/{sourceId}/remove-accounts` | Starts asynchronous removal of all accounts for the source; returns a task reference with an `id`. |
| Remove one account | `POST /accounts/v1/{accountId}/remove` | Starts asynchronous removal of one account from ISC without provisioning changes to the source; it can return on a later aggregation. Use only for a confirmed, specific account. The returned task ID may not resolve through `/task-status/v1/{id}`; verify the account directly. See the observed blocker note below. |
| Reset entitlements | `POST /entitlements/v1/reset/sources/{sourceId}` | Starts asynchronous removal of all entitlements for the source; returns a task reference with an `id`. |
| Poll either task | `GET /task-status/v1/{taskId}` | Returns task status. Poll until `completionStatus` is no longer `null`. |

The VS Code extension's `Reset source` implementation calls the reset-accounts endpoint first and calls reset-entitlements only when the account task's `completionStatus` is exactly `Success`. If the account reset warns, fails, is cancelled while being monitored, or returns no completed task, the extension does not continue to the entitlement reset.

The extension polls task status every five seconds. Its progress cancellation stops the local wait/polling; it does not call a server-side task-cancel endpoint, so do not assume the ISC job itself was cancelled.

## Shared task monitoring

Use the variables set at the top of this runbook on every `sail` command. Each reset POST returns a task ID. Capture and verify the returned `id`, then poll it:

```bash
sail --env "$ISC_ENV" api get "/task-status/v1/$TASK_ID" --pretty
```

Repeat about every five seconds while `completionStatus` is `null`. Continue only on `Success`. For `Warning` or any failure status, stop and inspect the returned `messages` and task details before deciding what to do next. Do not blindly retry a reset after a timeout: first query the original task ID to determine whether the asynchronous operation is still running or completed.

## Reset accounts only

This corresponds to the extension's **Reset accounts** command.

```bash
sail --env "$ISC_ENV" api post "/sources/v1/$SOURCE_ID/remove-accounts" \
  --body-file "$EMPTY_BODY_FILE" --pretty
```

1. Confirm the environment, source name, and source ID immediately before the POST.
2. Submit the POST once and capture its returned task `id`.
3. Poll `GET /task-status/v1/{taskId}` as described above.
4. Verify the source's accounts are absent from ISC after successful completion.

The API's documented behavior is to remove accounts from ISC without provisioning changes to the source. Those accounts may be created again by a later aggregation. This command does **not** reset entitlements.

### Observed blocker: source-owner accounts

In live testing on a non-production source, `POST /sources/v1/{sourceId}/remove-accounts` returned HTTP 400 with cause `Source Owners account exists on source` and listed the blocking account IDs. No bulk-reset task was started. The IDs were matched to source-owner identities that had accounts on that source.

After the user explicitly authorized removal of those specific accounts, each was removed one at a time with `POST /accounts/v1/{accountId}/remove`. In this observation, each call returned 202; the dashed task IDs did not resolve through `/task-status/v1/{id}` or `/beta/task-status/{id}`, so each removal was verified by a direct account lookup returning 404. The bulk reset-accounts operation then completed with `Success`, followed by the entitlement reset. Final source-scoped account and entitlement queries were empty.

Treat this as a **verified, tenant/source-specific recovery path**, not a general rule that every owner account is protected or that single-account removal always bypasses the bulk-reset restriction. Before using it, re-fetch the current source owners and source-scoped accounts, match only the exact blocked IDs, explain that each account will be removed from ISC and may return on aggregation, and obtain explicit user confirmation for those individual removals. If any per-account request is rejected, its result is ambiguous, or the account remains, stop and investigate; do not continue bulk deletion blindly. Once the blocker accounts are verified absent, retry the bulk reset-accounts operation and proceed to entitlement reset only if that task succeeds.

## Reset entitlements only

This corresponds to the extension's **Reset entitlements** command.

```bash
sail --env "$ISC_ENV" api post "/entitlements/v1/reset/sources/$SOURCE_ID" \
  --body-file "$EMPTY_BODY_FILE" --pretty
```

1. Confirm the environment, source name, and source ID immediately before the POST.
2. Submit the POST once and capture its returned task `id`.
3. Poll `GET /task-status/v1/{taskId}` as described above.
4. Verify that the source's entitlement objects are absent after successful completion.

This command does **not** reset account objects. To reload account data together with the reset entitlements, the API client documentation calls for an **unoptimized account aggregation** (`disableOptimization=true`). That is a separate operation; the extension's reset-entitlements command does not start it. Use the [source aggregation runbook](source-aggregation.md) to run the source's account aggregation with optimization disabled, and wait for that aggregation task to finish.

## Reset source (accounts, then entitlements)

This corresponds to the extension's **Reset source** command. It asks for confirmation, resets accounts, waits for that task, and resets entitlements only if the account reset completed successfully.

For a manual `sail` run, preserve that order:

1. Confirm the selected environment, source name, and source ID. Warn the operator that both ISC account objects and entitlement objects for the source will be removed.
2. Submit the reset-accounts POST from the previous section.
3. Poll its task to completion. Proceed only when `completionStatus` is `Success`.
4. Submit the reset-entitlements POST from the previous section.
5. Poll its task to completion. Investigate warnings or failures; do not report the full reset as successful unless both tasks completed successfully.
6. If the source is to be restored, run an unoptimized account aggregation (`disableOptimization=true`) as a separate step and verify that accounts and entitlements are repopulated. The reset-source command itself does not trigger aggregation.

Do not parallelize the account and entitlement reset calls. If account reset fails or finishes with a warning, stop before resetting entitlements; that matches the extension's sequencing and avoids leaving a partially reset source without first understanding the failure.

## Verification and recovery

- Check each asynchronous task's final `completionStatus` and `messages`; an accepted POST only means a job was started, not that the reset succeeded.
- After reset, inspect source-scoped accounts and entitlements using the corresponding ISC APIs/UI. Keep source ID filters explicit; large lists may require pagination.
- After repopulation, verify account aggregation and entitlement aggregation outcomes separately. For a specific account, compare its account `attributes` with `GET /accounts/v1/{accountId}/entitlements`; an account attribute containing source values is not proof that entitlement objects have been persisted and associated.
- If a POST response is lost, do not issue another reset immediately. Recover the task ID from available task/audit evidence or inspect the source state first.

## Implementation references

Reviewed in `yannick-beot-sp/vscode-sailpoint-identitynow` at commit `c7a259a` (extension `1.3.33`; pinned `sailpoint-api-client` `2.1.28`):

- [`TreeManager.ts`: reset accounts and entitlements](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/services/TreeManager.ts#L48-L105)
- [`TreeManager.ts`: full reset sequencing](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/services/TreeManager.ts#L296-L310)
- [`ISCClient.ts`: reset endpoint methods](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/services/ISCClient.ts#L437-L455)
- [`sourceUtils.ts`: task polling and completion handling](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/commands/source/sourceUtils.ts#L6-L29)
