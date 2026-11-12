# SaaS connectivity customizers

A customizer intercepts a compatible connector's input/output; it does not implement a new connector. Use the shared [SDK](sdk.md), [discovery workflow](discovery.md) and [spec template](spec-template.md), plus the vendor [overview](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers). The overview states Quick Compliance connectors are not supported; verify current compatibility and underlying connector behavior for the intended source/version.

## Discover before designing

Identify underlying connector family/version, source/environment and the exact problem. Confirm whether the source already supports the requirement, what before/after hooks or named customized operations it exposes, exact inputs/outputs, execution order, configuration and permissible changes. Obtain representative **sanitized** payloads or types; do not assume a target object's fields or a proprietary connector's internal implementation. Document added calls, auth, latency, sensitive data and failure impact. If support is unknown, mark the slice blocked rather than inventing a hook.

## SDK shape and hook contracts

The vendor [full example](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/example) exports `connectorCustomizer` and creates a customizer with `createConnectorCustomizer()`. Before handlers accept a typed command input and return the input type; after handlers accept and return the output type. Check the specific handler signature in the installed SDK. Preserve untouched fields, valid keys and required structures; define handling for missing/null data and errors. For list hooks test behavior per emitted item, ordering and aggregation overhead, rather than assuming one callback sees a whole array.

The documented hooks include [account create](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-create), [delete](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-delete), [disable](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-disable), [enable](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-enable), [list](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-list), [read](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-read), [unlock](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-unlock), [update](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/account-update), [password](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/change-password), [entitlement list](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/entitlement-list), [entitlement read](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/entitlement-read), [source data discover](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/source-data-discover), [source data read](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/source-data-read), and [test connection](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/commands/test-connection). The SDK may export additional hooks; check support for the specific connector and platform, not just type presence.

A [customized operation](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/customized-operation) is named and invoked from connector code, not an automatic before/after hook. Require evidence that the connector calls the exact identifier and document its input/output. Do not assume an out-of-the-box connector exposes a named operation. SailPoint documents endpoint-level named operations specifically for Web Services SaaS; see [Web Services SaaS endpoint customizers](../generic/web-services-saas.md#endpoint-level-customizers) and verify the endpoint flags, linked customizer and deployed handler.

## Illustrative after-read handler

This synthetic transformation only demonstrates the type and preservation of untouched fields; it is not a recommendation to change real display names. Implement it only if the exact hook, field and transformation are supported and approved for the connector.

```typescript
import { createConnectorCustomizer } from '@sailpoint/connector-sdk'

export const connectorCustomizer = async () => createConnectorCustomizer()
    .afterStdAccountRead(async (_context, output) => {
        const displayName = output.attributes.displayName
        if (typeof displayName !== 'string') return output
        return {
            ...output,
            attributes: { ...output.attributes, displayName: displayName.trim() },
        }
    })
```

Test the intended value, missing/non-string input, other attributes and key/status preservation. A type-correct callback does not establish that the underlying connector invokes it.

## Test and lifecycle

Write isolated tests for input/output mutation, untouched fields, nulls, errors, authentication, any external calls and repeated invocation. Where appropriate, the [vendor testing page](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/testing) illustrates testing a customizer alongside a custom connector locally; this does not prove behavior for a built-in deployed connector. Use synthetic fixtures, safe logging and the actual SDK.

[Getting started](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/getting-started), [build/upload](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/upload) and [linking](https://developer.sailpoint.com/docs/connectivity/saas-connectivity/customizers/linking) describe the lifecycle. Creating, uploading, linking or unlinking changes a tenant/source: require explicit authorization for exact environment, source, customizer version and action, and an agreed rollback. Confirm installed CLI flags and session environment first; do not run examples as an implicit part of discovery.
