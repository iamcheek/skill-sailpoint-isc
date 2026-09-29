# Workflow Execution Searching

Use this reference when you need to find a particular trigger input inside the execution history of an ISC workflow.

This is separate from searching the `events` index: the workflow execution APIs contain the actual workflow execution detail, including the trigger input captured by the execution.

## Endpoints

List executions for a workflow:

```text
GET /workflows/v1/{workflowId}/executions
```

Fetch one execution:

```text
GET /workflow-executions/v1/{executionId}
```

Fetch execution history:

```text
GET /workflow-executions/v1/{executionId}/history
GET /workflow-executions/v1/{executionId}/history-v2
```

Prefer `history-v2` when looking for the trigger input. In live responses, the useful input is under history events such as:

```text
history[*].attributes.input
history[*].attributes.input.trigger
history[*].attributes.input.trigger.identity.id
```

Depending on the execution and event type, the same trigger input may be repeated in several history events. Treat the execution ID—not each repeated history occurrence—as the unique match.

> The top-level `trigger` in the `history-v2` response describes the workflow definition's trigger. It is not necessarily the runtime trigger payload. To find the identity or other values that caused a particular execution, inspect the history event input, especially `history[*].attributes.input.trigger`.

## Find executions in a date range

The history endpoints do not accept date-range filters. Filter the workflow's execution list first, then fetch history for the returned execution IDs.

The execution-list endpoint supports `start_time` comparisons (`eq`, `lt`, `le`, `gt`, and `ge`):

```bash
sail --env <env> api get \
  '/workflows/v1/<workflow-id>/executions' \
  --query 'filters=start_time ge "<start-time>" and start_time lt "<end-time>"' \
  --query 'limit=250' \
  --query 'offset=0' \
  --jsonpath '$'
```

Workflow executions are retained for up to 90 days. The normal collection page size is at most 250; paginate with `offset` and continue until a page is empty or shorter than the requested limit.

Use `--jsonpath '$'` when capturing a response for a script. `--pretty` splits the JSON response and status/log output across streams and is intended for human-readable inspection, not reliable parsing.

## Fetch one execution by ID

For a known execution ID:

```bash
sail --env <env> api get \
  '/workflow-executions/v1/<execution-id>/history-v2' \
  --jsonpath '$' > /tmp/workflow-history.json

jq '.' /tmp/workflow-history.json
```

A single detail response can be large. Avoid using `tail` or manually stripping pretty-printed output when the goal is to search it; capture the clean JSON response and parse it structurally.

## Find a specific identity in trigger input

The following Python example searches every history event's input for a target identity ID and prints the execution metadata plus the matching trigger changes:

```python
import json
import subprocess

WORKFLOW_ID = "<workflow-id>"
TARGET_ID = "<identity-id>"


def get_json(path, queries=()):
    command = ["sail", "--env", "<env>", "api", "get", path]
    for query in queries:
        command += ["--query", query]
    command += ["--jsonpath", "$"]
    result = subprocess.run(
        command,
        check=True,
        capture_output=True,
        text=True,
    )
    return json.loads(result.stdout)


def find_trigger_inputs(value, path=""):
    matches = []
    if isinstance(value, dict):
        identity = value.get("identity")
        if isinstance(identity, dict) and identity.get("id") == TARGET_ID:
            matches.append({
                "path": path,
                "identity": identity,
                "changes": value.get("changes"),
            })
        for key, child in value.items():
            matches.extend(find_trigger_inputs(child, f"{path}/{key}"))
    elif isinstance(value, list):
        for index, child in enumerate(value):
            matches.extend(find_trigger_inputs(child, f"{path}/{index}"))
    return matches

executions = get_json(
    f"/workflows/v1/{WORKFLOW_ID}/executions",
    [
        'filters=start_time ge "<start-time>" and '
        'start_time lt "<end-time>"',
        "limit=250",
        "offset=0",
    ],
)

for execution in executions:
    history = get_json(
        f"/workflow-executions/v1/{execution['id']}/history-v2"
    )
    matches = find_trigger_inputs(history)
    if matches:
        print(json.dumps({
            "execution": execution,
            "matches": matches,
        }, indent=2))
```

In one workflow investigation, this found the runtime trigger input at paths such as:

```text
/history/0/attributes/input
/history/4/attributes/input/trigger
```

The important point is that searching only for a field named `triggerInput` can produce a false negative. The runtime payload may be nested under `attributes.input` or `attributes.input.trigger` instead.

## Worked pattern: date-windowed execution search

For a workflow with a known ID, the reliable sequence is:

1. List executions for the desired UTC date range:
   ```bash
   sail --env <env> api get \
     '/workflows/v1/<workflow-id>/executions' \
     --query 'filters=start_time ge "<start-time>" and start_time lt "<end-time>"' \
     --query 'limit=250' \
     --jsonpath '$'
   ```
2. Fetch `history-v2` for each returned execution.
3. Recursively inspect history event inputs for the identity ID, rather than searching only the top-level definition trigger.
4. Report the execution ID, execution start time, and the matching `changes` array.

The first lifecycle transition in the investigation did not produce a matching workflow execution because the workflow filter used `newlyterminated` while the event value was `newlyTerminated`; trigger filter comparisons should be treated as case-sensitive. The later `fullyTerminated` transition did match because the casing was identical.

## Workflow definition versus runtime input

To inspect the workflow's configured trigger filter, fetch the workflow definition:

```bash
sail --env <env> api get \
  '/workflows/v1/<workflow-id>' \
  --jsonpath '$.trigger' \
  --pretty
```

A definition may contain a filter such as:

```text
$.changes[?(@.attribute == "cloudLifecycleState" &&
  (@.newValue == "newlyterminated" ||
   @.newValue == "fullyTerminated"))]
```

Compare those literal values—including capitalization—with the runtime event's `changes[].newValue`. A trigger filter that differs only by case may not match.

## Cross-checking with the events index

The workflow history is the authoritative source for the input to a particular workflow execution. The `events` index is useful for finding the corresponding platform event or checking the state transition independently:

```bash
sail --env <env> api post '/search/v1' \
  --body '{
    "indices": ["events"],
    "query": {
      "query": "target.name:<identity-name> AND attributes.attributeValue:<new-state>"
    },
    "limit": 250
  }' \
  --pretty
```

For identity lifecycle changes, useful fields include:

```text
technicalName: IDENTITY_STATE_CHANGE_PASSED
attributes.attributeName: cloudLifecycleState
attributes.oldState
attributes.newState
attributes.attributeValue
```

An events search may identify the platform event even when no workflow execution exists—especially when the workflow's trigger filter did not match the event exactly.
