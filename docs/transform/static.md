# Static

**Transform type:** `"static"`

The static transform returns a fixed string value or evaluates Velocity Template Language (VTL) expressions. It is commonly used for conditional logic (if/then/else) and loops, and accepts dynamically provided variables as inputs.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `value` | Yes | string | Fixed string or VTL expression to evaluate. |
| (dynamic variables) | No | object | Additional properties within `attributes` that define nested transforms (or static values); these can be referenced from within the `value` VTL expression by name (e.g. `$workerType`). |

Top-level (outside `attributes`):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `static`. |
| `name` | Yes | string | Display name in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether to reevaluate during nightly identity refresh. Default: `false`. |

## Example(s)

Example 1 — dynamic variable reference (returns the value of the nested `accountAttribute` transform via `$workerType`):
```json
{
  "attributes": {
    "workerType": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "empType"
      },
      "type": "accountAttribute"
    },
    "value": "$workerType"
  },
  "type": "static",
  "name": "Static Transform"
}
```

Example 2 — conditional logic using VTL `#if`/`#else`/`#end`:
```json
{
  "attributes": {
    "workerType": {
      "attributes": {
        "sourceName": "HR Source",
        "attributeName": "empType"
      },
      "type": "accountAttribute"
    },
    "value": "#if($workerType=='Employee')Full-Time#{else}Contingent#end"
  },
  "type": "static",
  "name": "Static Transform"
}
```

## Notes

- When used in create profiles, values can be entered directly in the input field.
- Attributes can be ordered so that higher-listed attributes generate values usable by lower-listed attributes.
- Supports Velocity Template Language (VTL) syntax, including conditionals and loops.
- Dynamic variables are defined as additional properties within `attributes` and can reference other transforms.
