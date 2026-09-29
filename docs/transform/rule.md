# Rule

**Transform type:** `"rule"`

The rule transform enables reuse of logic from Transform rules, allowing execution of code that is either not possible through transforms alone or is too complex to maintain with Seaspray (SailPoint's transform configuration language).

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `name` | Yes | string | The name of the Transform rule to invoke. |
| (dynamic variables) | No | various | Additional properties in the `attributes` object are passed to the Transform rule as static strings or as the results of nested transforms, and are available to the rule for calculation. |

Top-level (outside `attributes`):

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be set to `rule`. |
| `name` | Yes | string | Display name appearing in UI dropdown menus. |
| `requiresPeriodicRefresh` | No | boolean | Whether transform logic should be reevaluated during nightly identity refresh. Default: `false`. |

## Example(s)

Basic example — invoking a Transform rule with no extra variables:
```json
{
  "attributes": {
    "name": "Generate Random Number"
  },
  "type": "rule",
  "name": "Rule Transform"
}
```

Complex example — invoking a Transform rule with dynamic variables, including a nested transform (`input`) and static values (`operation`, `numChars`):
```json
{
  "attributes": {
    "name": "Cloud Services Deployment Utility",
    "operation": "getEndOfString",
    "input": {
      "type": "trim"
    },
    "numChars": "3"
  },
  "type": "rule",
  "name": "Rule Transform"
}
```

## Notes

- Variables defined in the `attributes` object (other than `name`) are passed to and available within the invoked Transform rule for calculation purposes.
