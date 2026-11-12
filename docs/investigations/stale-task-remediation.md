# Stale Task Remediation

Use this playbook when an ISC operation—such as updating an Identity Profile or resetting a source—is blocked because Identity Security Cloud reports identity/background tasks as still in progress.

> **High-risk manual correction.** A task with `completionStatus: null` may still be executing. Do not clear it just because it is old, blocks another operation, or appears in a pending-task list. A PATCH changes ISC's task record; it does not prove or undo the underlying identity/source work.

The SailPoint Discuss thread linked below describes a community workaround: list pending tasks, then PATCH a task's `completed` and `completionStatus` fields. The thread's bulk PowerShell example marks many tasks `SUCCESS` and excludes a task family by name. **Do not copy or run that bulk script.** It can conceal unfinished work, and filtering out one task name does not make the remaining tasks safe to complete.

## Endpoints

The current `sailpoint-api-client` task-management API documents:

| Purpose | Method and endpoint | Notes |
|---|---|---|
| List task status | `GET /task-status/v1` | Supports `completionStatus isnull` to find in-progress tasks, plus filters for `sourceId`, `type`, and `launcher`. The API client documents task records for up to 90 days. |
| Inspect one task | `GET /task-status/v1/{taskId}` | Inspect task type, source, timestamps, progress, messages, and completion state. |
| Update one task status | `PATCH /task-status/v1/{taskId}` | JSON Patch. The SDK documents updating `completionStatus` and `completed` to clear a pending task. This is a write operation; see the gates below. |

The 2024 Discuss thread uses the older beta paths `/beta/task-status/pending-tasks` and `/beta/task-status/{id}`. Prefer the current v1 task-management endpoints above when available. The beta pending-task endpoint may return `204 No Content` when empty; that behavior is noted in [the CLI reference](../cli/README.md#compact-json-inspection-and-204-responses).

## Read-only diagnosis

1. Select the intended ISC environment explicitly and identify the exact blocked operation, source ID, and any identity/task IDs in the error or UI.
2. List tasks with a null completion status. Filter by source when appropriate:
   ```bash
   # All in-progress task records visible through the v1 task-status API
   sail --env "$ISC_ENV" api get /task-status/v1 \
     --query 'filters=completionStatus isnull' --query 'limit=100' --pretty

   # In-progress tasks associated with a source
   sail --env "$ISC_ENV" api get /task-status/v1 \
     --query "filters=sourceId eq \"$SOURCE_ID\" and completionStatus isnull" \
     --query 'limit=100' --pretty
   ```
   Paginate with `offset` if necessary. Identity tasks may not have a `sourceId`; a source-only filter can therefore miss a global identity task.
3. For each plausible blocker, inspect its full record:
   ```bash
   sail --env "$ISC_ENV" api get "/task-status/v1/$TASK_ID" --pretty
   ```
   Confirm its task type, launcher, source/identity context (where present), `created`/`completed` timestamps, progress, and messages. Correlate with the ISC Monitor UI and relevant platform/connector logs when possible.
4. Do not treat age alone, a missing recent log line, or a large pending-task count as proof a task is stale. Confirm that the underlying work is no longer running and determine whether it completed, failed, or was abandoned.
5. If the task may still be active, its result is unknown, multiple task families are involved, or the blocker recurs, stop and contact SailPoint Support/the task owner. Do not reset the source or manually rewrite task status to bypass uncertainty.

### Compare with recent duration for the same target

When deciding whether an aggregation is unusually slow, compare it with recent completed tasks of the same type **and the same target**. Do not use an organization-wide average as a timeout. The v1 task API retains at most 90 days; paginate only as far back as useful. Some account-aggregation task records have `sourceId: null`, so filter the returned task objects by `target.id` rather than assuming `sourceId` is populated.

```bash
sail --env "$ISC_ENV" api get /task-status/v1 \
  --query 'filters=type eq "CLOUD_ACCOUNT_AGGREGATION"' \
  --query 'sorters=-created' --query 'limit=100' --jsonpath '$' \
  > /tmp/account-aggregation-history.json
```

For the matching `target.id`, calculate `completed - created` for completed tasks. Compare the median and an upper percentile for `SUCCESS` tasks; review `WARNING`/`ERROR` separately. A history query is diagnostic evidence, not a guarantee that the current task is stuck. This script prints duration statistics by completion status:

```bash
TARGET_ID='<target.id from the task record>'
python3 - "$TARGET_ID" /tmp/account-aggregation-history.json <<'PY'
import json, math, statistics, sys
from datetime import datetime

target_id, path = sys.argv[1:3]
tasks = json.load(open(path))
def minutes(value):
    return (datetime.fromisoformat(value[1].replace("Z", "+00:00")) -
            datetime.fromisoformat(value[0].replace("Z", "+00:00"))).total_seconds() / 60
for status in ("SUCCESS", "WARNING", "ERROR"):
    durations = sorted(minutes((t["created"], t["completed"])) for t in tasks
                       if (t.get("target") or {}).get("id") == target_id
                       and t.get("completed") and t.get("completionStatus") == status)
    if durations:
        p90 = durations[math.ceil(0.9 * len(durations)) - 1]
        print(f"{status}: n={len(durations)}, median={statistics.median(durations):.1f}m, "
              f"p90={p90:.1f}m, min={durations[0]:.1f}m, max={durations[-1]:.1f}m")
PY
```

**Observed example, not a service-level threshold:** in one non-production tenant, completed `SUCCESS` account-aggregation runs for a large VA-based source had a median of roughly 80 minutes and a 90th percentile near 90 minutes, yet one successful run took more than twice as long. `WARNING` runs were not necessarily longer. Exceeding the usual range alone is therefore not proof of a stale task; establish the baseline for the specific source and task type.

## Manual correction — only for a confirmed stale status record

Proceed only when evidence shows the underlying operation completed successfully and the task record alone remains pending. If the operation did not complete successfully or its outcome is unknown, **do not mark it `SUCCESS`**; escalate instead. A successful status correction is not a substitute for verifying the underlying account, entitlement, identity-profile, or source state.

Before the PATCH, obtain explicit user confirmation for the exact environment, task ID, task type, and intended status correction. Never infer permission from a request to diagnose the blocker or from approval to reset a source. For source resets, the separate [source-reset runbook](source-reset.md) still requires its own explicit confirmation for the source and reset scope.

For a confirmed successfully completed task, the API client documents this JSON Patch shape (use the actual current UTC time):

```json
[
  {"op":"replace","path":"/completed","value":"<YYYY-MM-DDTHH:MM:SSZ>"},
  {"op":"replace","path":"/completionStatus","value":"SUCCESS"}
]
```

Send the patch to the **single confirmed task ID**, not a list or bulk query:

```bash
sail --env "$ISC_ENV" api patch "/task-status/v1/$TASK_ID" \
  -H 'Content-Type: application/json-patch+json' \
  --body-file /tmp/confirmed-task-status-patch.json --pretty
```

Then re-read `GET /task-status/v1/{taskId}` and verify the recorded values. Preserve the task ID, pre-patch status, evidence for the underlying outcome, patch time, operator confirmation, and post-patch result. Re-check the original blocked operation only after the task record is verified; if it remains blocked, stop and investigate rather than clearing additional tasks blindly.

## Source material and limitations

- [SailPoint Discuss: “The system cannot ‘Update Identity Profile’ while Identity Tasks are in progress”](https://developer.sailpoint.com/discuss/t/the-system-cannot-update-identity-profile-while-identity-tasks-are-in-progress/76971) — community report and workaround, not an official recovery guarantee. Its bulk-success script is intentionally not reproduced as an executable recipe here.
- The endpoint and JSON Patch method above were cross-checked against `sailpoint-api-client` 2.1.28 task-management definitions: task list filters include `completionStatus isnull`; `updateTaskStatusV1` uses `PATCH /task-status/v1/{id}` and documents `completionStatus` plus `completed` for clearing a pending task. Verify the live API contract and tenant behavior before any manual status update.
