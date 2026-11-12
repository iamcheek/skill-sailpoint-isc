# Custom SaaS connector design and construction

Use for a connector implemented with the SailPoint connector SDK; do not confuse this with a built-in connector, a Web Services SaaS source or a [customizer](customizers.md). Start with [discovery](discovery.md) and [SDK foundations](sdk.md). Offline design needs neither `sail` nor a tenant.

## Project and runtime spec

The vendor's [prerequisites](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/prerequisites) describe `sail conn init <project>` and a TypeScript scaffold with a runtime `connector-spec.json`, source, configuration and build scripts. Inspect the actual scaffold and installed CLI help before use; don't assume a repository uses a particular layout. Creating local files or installing dependencies still requires normal project/host permission.

The [connector specification](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/connector-spec) describes name, key type, commands, source configuration, account schema, entitlement schemas and create template. Its account/create mappings do **not** establish the current live source policy or correlation. A repository design spec records requirements and decisions; the runtime `connector-spec.json` configures the connector. Check current source configuration separately when authorized.

For each item, define and verify:

1. **Target boundary and access:** target systems, endpoints/API versions, scopes, target populations, account and entitlement ownership, exclusions, source config fields and secret handling.
2. **Identifiers:** stable target ID, lookup key, SailPoint key/identity/UUID, correlation rules and behavior on duplicates, renames, missing keys and ambiguous matches. Do not equate target joins with ISC identity correlation.
3. **Schemas and mappings:** account and each entitlement type, native object types, required/optional/multivalued values, null versus empty and membership associations. Validate field names, schema relationships and actual platform support rather than assuming one schema named `group` means a target group.
4. **Operations:** declare only approved operations the connector actually handles; map each requirement to [command contracts](commands.md), target endpoints, auth, expected outputs and negative cases. Test declaration, registered handler and runtime policy together.
5. **Data and failure:** pagination, incremental state, large populations, concurrency, target consistency/rate limits, error mapping, safe retries, readback and partial-write recovery. Missing results must not silently become successful empty records.

Vendor docs may disagree across examples or versions: for example the reviewed SDK hello-world runtime spec omits fields that the current connector-spec article labels required. Verify the applicable project/tenant contract rather than generalizing either example. The article currently describes only `group` as an entitlement schema type; treat that statement as article-scoped and verify actual supported types and associations before making a broader claim.

## Illustrative handler shape

This synthetic simple-key example demonstrates the SDK seam; it is **not** a target implementation. Implement the client against an approved target contract, ensure `getAccount` returns `null` only for confirmed absence, and make the runtime spec, schema, output and tests match. The export shape shown by the target project's scaffold may differ.

```typescript
import { createConnector, SimpleKey, KeyID, ConnectorError, ConnectorErrorType } from '@sailpoint/connector-sdk'

type Account = { id: string; displayName: string; active: boolean }
interface TargetClient {
    testConnection(): Promise<void>
    listAccounts(): AsyncIterable<Account>
    getAccount(id: string): Promise<Account | null>
}

export function buildConnector(client: TargetClient) {
    const mapAccount = (a: Account) => ({
        key: SimpleKey(a.id),
        disabled: !a.active,
        attributes: { id: a.id, displayName: a.displayName },
    })
    return createConnector()
        .stdTestConnection(async (_context, _input, res) => {
            await client.testConnection()
            res.send({})
        })
        .stdAccountList(async (_context, _input, res) => {
            for await (const account of client.listAccounts()) res.send(mapAccount(account))
        })
        .stdAccountRead(async (_context, input, res) => {
            const account = await client.getAccount(KeyID(input))
            if (account === null) throw new ConnectorError('Account not found', ConnectorErrorType.NotFound)
            res.send(mapAccount(account))
        })
}
```

Test the registration and outputs using a synthetic client; separately test pagination and target error mapping in the real client. This example does not define correlation, provisioning, a runtime spec or a deployable connector.

## Development and handoff

Use [discovery and specification workflow](discovery.md) to gather decisions and produce a durable [implementation spec](spec-template.md); use [implementation and review](implementation-review.md) to build one ready slice. SDK declarations, target API documentation, known examples and tests are evidence, not user approval. Keep mock tests, controlled live checks, deployed version and stakeholder acceptance separate. For actual source creation/upload, obtain explicit approval for the exact environment/source/operation and use [runtime/testing](runtime-testing.md).
