# `sail` CLI Reference

Mechanics of the `sail` CLI tool itself — flags, output handling, and CLI-level gotchas. This is *how to invoke the tool*; for *what to call it with* (endpoints, filters, query syntax), see [docs/api/](../api/README.md) and [docs/search/](../search/README.md).

Every command in this file must include the explicitly selected `--env <name>` in practice — see the parent [SKILL.md](../../SKILL.md) Step 2a for environment selection, which is not repeated here since it's session setup, not a CLI mechanic. Preserve that flag on every command in a loop or script; never rely on the CLI's persisted active environment.

## `sail search query`

Writes results to a file in `search_results/` rather than printing them — you must read the file back after running the command:

```bash
sail search query "<QUERY>" --indices <INDEX> 2>&1
cat search_results/*.json | jq '.' 2>/dev/null
```

> **Gotcha (verified in live testing): a zero-hit query writes no file and prints no error/warning.** The command exits 0 and its log output looks identical to a successful query — no "0 results" message, nothing in `search_results/`. If you don't see a new file appear after running a query, that means zero hits, not a broken command or bad syntax — but there's no direct signal telling you that; it's inferred only from the file's absence. Don't spend time debugging query syntax before confirming this simple explanation first, and don't assume a missing file means the CLI failed.

See [docs/search/README.md](../search/README.md) for valid `<INDEX>` values and query syntax.

## `sail api {get,post,put,patch,delete}`

| Flag | Applies to | Notes |
|---|---|---|
| `--pretty` | all | Pretty-prints the JSON response. Always use this. |
| `--query "key=value"` | `get`, `put`, `patch`, `delete` — **not `post`** | Repeatable — pass once per query param (e.g. `--query "limit=50" --query "offset=0"`). **`sail api post` rejects this flag** with `Error: unknown flag: --query` (confirmed live). For POST requests, inline any query string directly into the path instead: `sail api post "/access-request-administration/v1?count=true&limit=1" ...`. |
| `--body '<json>'` | `post`, `put`, `patch` | Request body as a literal string. |
| `--body-file <path>` | `post`, `put`, `patch` | Request body read from a file — use for large bodies or non-JSON bodies. |
| `--content-type <type>` | `post`, `put`, `patch` | Sets the request body's Content-Type; POST/PUT default to `application/json`. Use with `--body`/`--body-file` for APIs requiring `multipart/form-data`. |
| `-H 'Header: value'` | all | Repeatable. Required for experimental endpoints, e.g. `-H 'X-SailPoint-Experimental: true'`. |
| `--jsonpath "$.field"` | all | Extract specific fields from the response instead of returning the full body. |

> **Gotcha (verified in live testing):** For `get`/`put`/`patch`/`delete`, don't inline a `filters=`/`sorters=` expression directly into the URL path — unlike `--query`, inlining does not URL-encode the value, so spaces and quotes go on the wire literally and the API returns 400. `sail api get "/accounts/v1?filters=nativeIdentity eq \"...\""` → 400 Bad Request; the identical filter via `--query 'filters=nativeIdentity eq "..."'` → 200 OK. Inlining into the path is only correct for `post`, which has no `--query` flag to fall back on.

### Examples

```bash
# GET with query params
sail api get /accounts/v1 --query "limit=50" --query "offset=0" --pretty

# POST — query params go in the path, not --query
sail api post "/search/v1?count=true" --body '{"indices":["identities"],"query":{"query":"*"},"limit":1}' --pretty

# Experimental endpoint header
sail api get /access-request-administration/v1 -H 'X-SailPoint-Experimental: true' --pretty

# JSONPath extraction
sail api get /accounts/v1 --query 'filters=identityId eq "abc123"' --jsonpath "$.id" --pretty
sail api get /identities/v1 --jsonpath "$[*].name" --pretty
```

## `sail conn logs`

Pulls connector logs for **SaaS-based connectors only** — virtual appliance (VA)-based connectors write logs locally on the VA, not here.

```bash
sail conn logs --start "<ISO8601-UTC>" --stop "<ISO8601-UTC>" 2>&1 > /tmp/conn_logs_<window>.txt
grep "<requestId>" /tmp/conn_logs_<window>.txt
```

Supports `--start`, `--stop`, `--target-id`, `--target-name`, `--level`, and `--request-id` flags. Use a tight window (±1-2 hours around a known UTC timestamp) rather than a full day — faster, and avoids huge files.

> **Verified in live testing:** `--request-id` and `--target-id` both filter server-side correctly — confirmed by comparing filtered vs. unfiltered output over the same window (e.g. `--request-id <id>` returned 108 lines vs. 13,093 unfiltered; `--target-id <id>` returned only that target's 3 lines). An older version of this doc warned the flag "may be silently ignored" — that did not hold in this test. If you ever see all logs returned regardless of `--request-id`, re-verify against a same-window unfiltered baseline before concluding the filter is broken again, rather than assuming it and grepping manually.
>
> **Separate gotcha that persists even with server-side filtering:** the requestId you're searching for may not appear as the top-level `requestID` field on the log line that has the detail you need. Coarse `invokeCommand`/`commandOutcome` lines carry it at the top level, but the detailed `connectorMessage`/`ERROR` lines with the actual stack trace can carry a *different* top-level `requestID` (the connector's own internal invocation id) — your target id only shows up nested inside `message.requestId` on those lines. When correlating, check both the top-level `requestID` field and the nested `message.requestId` field, not just one.
>
> This is about log-entry *structure* only — for whether `requestID` is queryable anywhere outside `sail conn logs`/`/platform-logs/v1/query` (it isn't — use `trackingNumber` to pivot to `accountactivities` instead), see [docs/investigations/provisioning.md](../investigations/provisioning.md#platform-logs-requestid-scope).

For the full provisioning-investigation use case this command supports, see [docs/investigations/provisioning.md](../investigations/provisioning.md).

## Output capture pattern

`sail --pretty` writes the JSON response to **stderr** and the HTTP status line to **stdout**. Always capture both:

```bash
sail api get /accounts/v1 --pretty 2>&1
```

To pipe into `jq`, strip the banner/log noise first — timestamps, `Status:` lines, and raw HTTP headers are interleaved with the JSON body:

```bash
sail api get /accounts/v1 --pretty 2>&1 | grep -v '^2026/' | grep -v '^Status:' | jq '.'
```

> **Default to `--jsonpath "$"` over `--pretty` whenever you intend to pipe the result into `jq`/parse it programmatically** — not just for pagination loops (see below). Repeatedly re-running `--pretty 2>&1 | tail -N` and manually stripping banner lines with `sed`/`grep` wastes turns when a stray redirect drops the body to the wrong stream (verified as a real time-sink in live testing: multiple `--pretty` calls redirected `2>&1` incorrectly and silently lost the JSON body, requiring re-runs). `--jsonpath "$"` (root — full body) avoids the split-stream problem entirely, since it's designed to be clean stdout output for scripting.

### Compact JSON inspection and 204 responses

When a response is a JSON array and only a count is needed, request the clean root value and let
`jq` count it. Keep the environment explicit:

```bash
sail --env <name> api get /accounts/v1 --jsonpath '$' 2>/dev/null | jq 'length'
```

Use the same pattern for other list endpoints, adding filters or pagination flags as needed. This
is preferable to parsing `--pretty` output or counting log text. If the endpoint returns a JSON
object rather than an array, inspect its relevant array field instead (for example,
`--jsonpath '$.items' | jq 'length'`).

Some no-results/task-status endpoints return **204 No Content** rather than `[]`. In particular,
`/beta/task-status/pending-tasks` may return 204 when there are no pending tasks. A blank body with
HTTP 204 is an expected “none” result, not malformed JSON; do not treat it as evidence that the
request failed or pipe the blank body into `jq` without handling the empty case. Use `--pretty` when
you need to see the status, or capture the status/output separately in a script.

When a clean JSONPath response is available, use it instead of repeatedly stripping banners from
`--pretty 2>&1`. Continue to use `--pretty` for human-readable, non-scripted API inspection.

**For scripted/multi-call use (e.g. paginating a loop), prefer `--jsonpath` over `--pretty` + grep/regex.** `--jsonpath "$.field"` writes just that field's value, on its own trailing line of stdout, with no surrounding banner noise to strip — take the last line rather than parsing the full mixed output:

```bash
# clean nextToken extraction, no regex needed
TOKEN=$(sail api post /platform-logs/v1/query -H 'X-SailPoint-Experimental: true' --body "$BODY" --jsonpath "$.nextToken" 2>/dev/null | tail -1)

# clean logs array extraction — directly parseable JSON
LOGS=$(sail api post /platform-logs/v1/query -H 'X-SailPoint-Experimental: true' --body "$BODY" --jsonpath "$.logs" 2>/dev/null | tail -1)
echo "$LOGS" | jq '.'
```

This avoids brace-counting or regex-scraping the `--pretty` output to isolate the JSON body — verified against `/platform-logs/v1/query` in live testing.

> **`sail conn logs` has no `--jsonpath` flag** (it's a separate subcommand from `sail api`, not one of the verbs above) — its raw output is one or more paginated JSON response blobs interleaved with the same banner/header noise, one blob per page. A regex like `grep -o '{"tenantID":...}}'` to pull out individual log entries will silently truncate or skip entries whose `message` field is itself nested JSON (e.g. `connectorMessage`/`ERROR` entries with a stack trace) — the non-greedy match stops at the first `}}` inside the nesting. Parse each page's JSON blob properly instead, e.g. Python's `json.JSONDecoder().raw_decode(text, pos)` in a loop over the blob start offsets, then filter/inspect the resulting `logs[]` objects — don't regex the raw text for individual log lines.
>
> **Separate, more basic truncation issue (verified in live testing): individual log lines themselves get hard-truncated at 4161 bytes**, cutting a long embedded JSON `message` field off mid-string before it ever reaches a parser — confirmed by measuring raw line lengths in a captured log file (hundreds of lines landed at exactly 4161 bytes). This bit a `std:account:update` "Running before account, for account ... provisioning plan is {...}" line, whose `changes`/`groupSchemas` payload is long enough to get cut off before the JSON closes — `json.loads()` on that line raises `Unterminated string`, not because the regex/nesting problem above applies, but because the source line was never complete to begin with. This is unrelated to the pagination-per-page truncation the note above describes — that one is about `grep`/regex extraction technique; this one is a hard cap on line length from the CLI itself, with no workaround other than extracting the fields you need (e.g. account ID, `changes` array) via a regex tolerant of a truncated tail, rather than requiring the full JSON blob to parse cleanly.

## Environment listing

`sail environment list` blocks on an interactive `Press Enter to continue` prompt — it reads stdin in a loop until it gets a non-empty line. Piping EOF (e.g. `< /dev/null`) causes it to loop instead of exiting. Always pipe a newline into it:

```bash
printf '\n' | sail environment list 2>&1
```

Full environment-selection workflow (why, when, and how to pick a tenant for the session) lives in the parent [SKILL.md](../../SKILL.md) Steps 1–2 — this is CLI-invocation mechanics only.

## CLI-level errors

| Error | Likely cause | Fix |
|---|---|---|
| `command not found: sail` | sail not installed | Install the sail CLI; if missing, tell the user and stop. |
| `Error: unknown flag: --query` | Used `--query` with `sail api post` (or another verb that doesn't support it) | Move the query string into the path itself, e.g. `sail api post "/path?key=value"`. |

For HTTP-level errors (400/401/403/404 from the API itself, as opposed to the CLI rejecting your invocation), see [docs/api/README.md](../api/README.md#common-errors).
