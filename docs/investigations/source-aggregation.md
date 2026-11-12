# Source Aggregation Runbooks

Use this playbook for the three source-level aggregation actions exposed by the community VS Code extension:

- **Aggregate source** — account aggregation with the connector's normal optimization behavior.
- **Aggregate source without optimization** — account aggregation with `disableOptimization=true`, which reprocesses every account whether or not its source data changed.
- **Aggregate entitlements** — entitlement/group aggregation for the source.

These are asynchronous ISC tasks. Starting a task is not proof that it completed successfully or that entitlements were persisted/associated as expected.

## Single-account reload

Use this when one known account needs to be reloaded directly from its connector and undergo a one-time aggregation. This is distinct from source-wide aggregation; do not substitute `POST /sources/v1/{sourceId}/load-accounts`.

1. Verify the target with `GET /accounts/v1/{accountId}`. Use its SailPoint account `id`—not the identity ID, source ID, or source-native identity.
2. The API requires scope `idn:accounts-state:manage`; supported user levels are `ORG_ADMIN`, `SOURCE_ADMIN`, `SOURCE_SUBADMIN`, and `HELPDESK`.
3. Submit the reload:

   ```bash
   ISC_ENV='<selected-environment>'
   ACCOUNT_ID='<SailPoint-account-id>'
   sail --env "$ISC_ENV" api post "/accounts/v1/$ACCOUNT_ID/reload" --body '{}' --pretty
   ```

   The OpenAPI operation has no request body, but `sail api post` requires `--body` or `--body-file`; `{}` was accepted in live testing.
4. The API returns `202` with an `id` for the asynchronous task. Treat this as accepted, not completed. In live testing, a returned dashed UUID was not found at `/task-status/v1/{id}`; the generic task-status route is therefore not verified for these reload IDs. Do not report completion based only on the `202` response.

## Preconditions and safety

1. Follow the SailPoint skill's environment-selection procedure and use `--env <name>` on every `sail` call.
2. Verify the source name and ID in that environment before starting the task.
3. Avoid starting duplicate or conflicting account/entitlement aggregations for the same source. Check active tasks first:
   ```bash
   sail --env "$ISC_ENV" api get /task-status/v1 --query "filters=sourceId eq \"$SOURCE_ID\" and completionStatus isnull" --pretty
   ```
4. The VS Code extension applies its tenant-read-only guard before starting any of these operations. Direct API calls do not invoke that local guard; get the appropriate operator approval before using them.
5. Entitlement aggregation requires a token with `ORG_ADMIN`, `SOURCE_ADMIN`, or `SOURCE_SUBADMIN` authority, according to the API client reference. Use a principal with the permissions required for the target source for account aggregation as well.

Set variables after choosing the environment:

```bash
ISC_ENV='<selected-environment>'
SOURCE_ID='<source-id>'
```

## API endpoints and task IDs

| Extension action | API request | Task ID in response |
|---|---|---|
| Aggregate source (accounts, normal optimization) | `POST /sources/v1/{id}/load-accounts` | `task.id` |
| Aggregate source without optimization | `POST /sources/v1/{id}/load-accounts` with multipart form field `disableOptimization=true` | `task.id` |
| Aggregate entitlements | `POST /sources/v1/{sourceId}/load-entitlements` | `id` |
| Poll task status | `GET /task-status/v1/{taskId}` | — |

The extension's SDK sends the account and entitlement aggregation requests as `multipart/form-data`. For the normal account aggregation it omits `disableOptimization`; for the no-optimization action it sends the multipart form field with the string value `true`. It does not send a JSON request body for these actions. For Delimited File sources, an optional CSV file is also sent; use the extension's file-import flow or another multipart-capable client for that case.

### Raw `sail` CLI request bodies

`sail api post` defaults to `application/json`, so set `--content-type` and send a multipart body rather than a JSON object. The following examples are for connector-backed sources without an uploaded CSV file. They create temporary, non-secret body files and use an explicit boundary.

For **Aggregate source** (normal optimization) or **Aggregate entitlements**, the extension sends an empty multipart form:

```bash
BOUNDARY="isc-aggregate-$(date +%s)"
printf -- '--%s--\r\n' "$BOUNDARY" > /tmp/isc-aggregate-empty.form
```

Use the corresponding endpoint:

```bash
# Normal account aggregation
sail --env "$ISC_ENV" api post "/sources/v1/$SOURCE_ID/load-accounts" \
  --content-type "multipart/form-data; boundary=$BOUNDARY" \
  --body-file /tmp/isc-aggregate-empty.form --pretty

# Entitlement aggregation
sail --env "$ISC_ENV" api post "/sources/v1/$SOURCE_ID/load-entitlements" \
  --content-type "multipart/form-data; boundary=$BOUNDARY" \
  --body-file /tmp/isc-aggregate-empty.form --pretty
```

For **Aggregate source without optimization**, send the `disableOptimization=true` form field:

```bash
BOUNDARY="isc-aggregate-$(date +%s)"
printf -- '--%s\r\nContent-Disposition: form-data; name="disableOptimization"\r\n\r\ntrue\r\n--%s--\r\n' \
  "$BOUNDARY" "$BOUNDARY" > /tmp/isc-aggregate-unoptimized.form

sail --env "$ISC_ENV" api post "/sources/v1/$SOURCE_ID/load-accounts" \
  --content-type "multipart/form-data; boundary=$BOUNDARY" \
  --body-file /tmp/isc-aggregate-unoptimized.form --pretty
```

Inspect the response and capture the task ID. Do not substitute a JSON body such as `{"disableOptimization":true}`; the extension and generated API client send this as a multipart form field.

## Shared task monitoring

Poll the returned task ID about every five seconds, as the extension does:

```bash
sail --env "$ISC_ENV" api get "/task-status/v1/$TASK_ID" --pretty
```

Continue while `completionStatus` is `null`. Treat only `Success` as a successful completion. For `Warning` or any failure status, inspect `messages` and the task details before retrying or moving to a dependent operation.

The extension's progress UI is cancellable, but its polling helper only stops waiting when cancellation is requested; it does not call a server-side task-cancel endpoint. If progress monitoring is cancelled or interrupted, query the original task ID before starting another aggregation.

## Routine 1: Aggregate accounts (normal optimization)

1. Confirm the environment and source ID/name; check for active source tasks.
2. Submit `POST /sources/v1/{id}/load-accounts` with an empty multipart form and no `disableOptimization` field (or use the extension's **Aggregate source** action).
3. Capture the returned `task.id` and poll `/task-status/v1/{taskId}` to completion.
4. On `Success`, verify the resulting account data and any required account processing. On warning/failure, inspect the task messages before retrying.

This uses the endpoint's normal/default optimization behavior. It is the routine account aggregation, not a force-reprocess request.

## Routine 2: Aggregate accounts without optimization

1. Confirm the environment and source ID/name; check for active source tasks.
2. Submit the same account endpoint with multipart field `disableOptimization=true` (or use the extension's **Aggregate source without optimization** action).
3. Capture the returned `task.id` and poll it to completion.
4. On `Success`, verify accounts and their post-aggregation attributes/entitlement associations. Inspect warnings and task messages before retrying if it does not succeed.

Use this when every account must be reprocessed even if its data did not change—for example, after correcting account mappings or when deliberately rebuilding account-to-entitlement associations. This can be more expensive than the normal aggregation; do not choose it by default.

## Routine 3: Aggregate entitlements

1. Confirm the environment and source ID/name; check for active source tasks.
2. Submit `POST /sources/v1/{sourceId}/load-entitlements` with an empty multipart form for connector-backed sources (or use **Aggregate entitlements** in the extension). Delimited File sources may require the CSV file.
3. Capture the returned `id` and poll `/task-status/v1/{taskId}` to completion.
4. On `Success`, verify entitlement objects are persisted for the source. A successful task or connector scan alone does not prove account associations were created.

Entitlement aggregation is separate from account aggregation. If accounts reference group entitlements, aggregate entitlements first, then run account aggregation so the account values can resolve to the aggregated entitlement objects. When correcting existing account associations, consider the unoptimized account routine after the entitlement aggregation.

## Rebuild after a source reset

A source reset removes imported data and does not launch aggregations. To repopulate a source whose account schema references group entitlements, the usual sequence is:

1. Aggregate entitlements and wait for `Success`.
2. Aggregate accounts without optimization and wait for `Success`.
3. Verify source-scoped entitlement objects, account attributes, and account entitlement associations independently.

For reset semantics and the reset endpoints, see [source-reset.md](source-reset.md).

## Verification guidance

- Confirm the task's final `completionStatus` and inspect `messages`; a `202`/accepted task response is only task creation.
- **Verify three stages separately, for any connector.** A connector returning an object does not mean ISC persisted it or associated it with an account:
  1. **Connector scan:** what the target returned and what the connector reports as scanned (for example, an HTTP 200 or an `Entitlements: N` log line).
  2. **Persistence:** what an entitlement or account endpoint and the ISC UI can retrieve. For newly aggregated entitlements, a zero result or 404 on `/entitlements/v1` is not proof of non-persistence; check the beta path described in [endpoints](../api/endpoints.md#beta-entitlement-endpoints).
  3. **Association:** whether the account's entitlement attribute contains the entitlement's identity, and what `/accounts/v1` reports.
- **Scanned count is not unique-account count.** When diagnosing duplicates or missing objects, compare the number of unique native identities after aggregation with the connector's scanned count. Unstable ordering between pages can produce duplicates and omissions with no error.
- Check account records and entitlement objects separately. `attributes.<entitlementAttribute>` on an account does not itself prove that persisted entitlement records are associated to that account.
- For a specific account, compare its `/accounts/v1/{accountId}` attributes with `/accounts/v1/{accountId}/entitlements`; inspect the source-scoped entitlement list separately.
- On Web Services SaaS sources, also verify the configured response root, mappings, group schema/account-schema reference, and aggregation order. See [web-services-saas.md](../connectors/generic/web-services-saas.md).

## Implementation references

Reviewed in `yannick-beot-sp/vscode-sailpoint-identitynow` at commit `c7a259a` (extension `1.3.33`; pinned `sailpoint-api-client` `2.1.28`):

- [`package.json`: extension command labels](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/package.json#L220-L238)
- [`extension.ts`: command routing](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/extension.ts#L190-L220)
- [`TreeManager.ts`: account and entitlement aggregation](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/services/TreeManager.ts#L108-L137)
- [`TreeManager.ts`: account aggregation and optimization option](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/services/TreeManager.ts#L270-L293)
- [`ISCClient.ts`: aggregation API client calls](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/services/ISCClient.ts#L408-L478)
- [`sourceUtils.ts`: task polling and completion handling](https://github.com/yannick-beot-sp/vscode-sailpoint-identitynow/blob/c7a259a/src/commands/source/sourceUtils.ts#L6-L29)
