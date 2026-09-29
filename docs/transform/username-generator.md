# Username Generator

**Transform type:** `"usernameGenerator"`

Specifies logic for deriving unique attribute values (e.g. usernames, account IDs, distinguished names) during account creation. It combines identity data (such as name parts, built from nested transforms) and can automatically increment a counter (`$uniqueCounter`) to guarantee uniqueness across the target system.

## Attributes

| Attribute | Required | Type | Description |
|---|---|---|---|
| `patterns` | Yes | array of strings | Sequential patterns the generator evaluates, in order, to find a unique value. Supports variable substitution (e.g. `$fn`, `$ln`) referencing other keys defined in `attributes`, and the special token `${uniqueCounter}` for automatic incrementing. |
| `sourceCheck` | No | boolean | When `true`, the generator queries the target system directly (if supported) to check for uniqueness; when `false`, it checks only the Identity Security Cloud database. Default: `false`. |
| *(dynamic keys)* | No | transform object | Any additional properties in the `attributes` object act as substitutable variables referenced by name in `patterns` (e.g. `fn`, `ln`, `fi`, `mi`). Each is itself a nested transform (e.g. `identityAttribute`, `substring`). |

### Required properties on the containing account attribute (outside the transform)

| Property | Required | Type | Description |
|---|---|---|---|
| `cloudMaxSize` | Yes | string | Maximum character length allowed before truncation. |
| `cloudMaxUniqueChecks` | Yes | string | Maximum number of uniqueness-check iterations the generator will attempt (maximum allowed value is 50). |
| `cloudRequired` | Yes | string | Must be `"true"`. |

### Top-level transform property

| Property | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | string | Must be `"usernameGenerator"`. |

## Important constraints

- `${uniqueCounter}` must always be the **last** element in a pattern — the generator stops processing further pattern segments once it encounters `uniqueCounter`.
- If no unique value can be found within the `cloudMaxUniqueChecks` limit, an `IllegalStateException` is thrown.

## Example(s)

First initial + last name with counter, using a nested `substring` transform to derive the first initial:

```json
{
  "name": "userId",
  "transform": {
    "type": "usernameGenerator",
    "attributes": {
      "sourceCheck": true,
      "patterns": ["$fi$ln${uniqueCounter}"],
      "ln": {
        "type": "identityAttribute",
        "attributes": {
          "name": "lastname"
        }
      },
      "fi": {
        "type": "substring",
        "attributes": {
          "input": {
            "type": "identityAttribute",
            "attributes": {
              "name": "firstname"
            }
          },
          "begin": 0,
          "end": 1
        }
      }
    }
  },
  "attributes": {
    "cloudMaxSize": "100",
    "cloudMaxUniqueChecks": "25",
    "cloudRequired": "true"
  },
  "isRequired": false,
  "type": "string",
  "isMultiValued": false
}
```

First name + period + last name with counter:

```json
{
  "name": "accountId",
  "transform": {
    "type": "usernameGenerator",
    "attributes": {
      "sourceCheck": true,
      "patterns": ["$fn.$ln${uniqueCounter}"],
      "fn": {
        "type": "identityAttribute",
        "attributes": {
          "name": "firstname"
        }
      },
      "ln": {
        "type": "identityAttribute",
        "attributes": {
          "name": "lastname"
        }
      }
    }
  },
  "attributes": {
    "cloudMaxSize": "100",
    "cloudMaxUniqueChecks": "10",
    "cloudRequired": "true"
  },
  "isRequired": false,
  "type": "string",
  "isMultiValued": false
}
```

Distinguished name pattern with multiple fallback patterns (first tries first-initial.lastname, then firstname.lastname, then firstname.middleinitial.lastname+counter):

```json
{
  "name": "distinguishedName",
  "transform": {
    "type": "usernameGenerator",
    "attributes": {
      "sourceCheck": true,
      "patterns": [
        "CN=$fi.$ln,OU=Users,DC=YourDomain,DC=com",
        "CN=$fn.$ln,OU=Users,DC=YourDomain,DC=com",
        "CN=$fn.$mi.$ln${uniqueCounter},OU=Users,DC=YourDomain,DC=com"
      ],
      "fn": {
        "type": "identityAttribute",
        "attributes": {
          "name": "firstname"
        }
      },
      "ln": {
        "type": "identityAttribute",
        "attributes": {
          "name": "lastname"
        }
      },
      "fi": {
        "type": "substring",
        "attributes": {
          "input": {
            "type": "identityAttribute",
            "attributes": {
              "name": "firstname"
            }
          },
          "begin": 0,
          "end": 1
        }
      },
      "mi": {
        "type": "substring",
        "attributes": {
          "input": {
            "type": "identityAttribute",
            "attributes": {
              "name": "middlename"
            }
          },
          "begin": 0,
          "end": 1
        }
      }
    }
  },
  "attributes": {
    "cloudMaxSize": "100",
    "cloudMaxUniqueChecks": "5",
    "cloudRequired": "true"
  },
  "isRequired": false,
  "type": "string",
  "isMultiValued": false
}
```

## Notes

- `sourceCheck: true` requires the connector for the target system to support direct uniqueness lookups; otherwise uniqueness is checked only against the Identity Security Cloud database.
- `cloudMaxUniqueChecks` has a hard maximum of 50.
- This transform is typically used inside an account attribute's `transform` definition (nested under a schema attribute config with `cloudMaxSize`, `cloudMaxUniqueChecks`, and `cloudRequired`), not as a standalone identity attribute transform.
