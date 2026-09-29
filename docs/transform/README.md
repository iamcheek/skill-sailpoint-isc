# SailPoint Transforms

Transforms are JSON-configured "building blocks" used in SailPoint Identity Security Cloud (ISC) to manipulate attribute values during identity aggregation and account provisioning — without writing code. Internally SailPoint has also referred to this feature by the codename **Seaspray**.

This directory is a reference set for working with transforms: this file is the conceptual map, and each transform operation has its own file with exact attribute names, types, and examples.

- Source docs: https://developer.sailpoint.com/docs/extensibility/transforms and https://developer.sailpoint.com/docs/extensibility/transforms/operations
- Provisioning-policy-specific guidance: [provisioning-policies.md](provisioning-policies.md)
- Each operation file lives at `docs/transform/<slug>.md` (see the [Operation Index](#operation-index) below).

---

## Core concepts

### Structure

A transform is a JSON object with three parts:

```json
{
  "name": "Lowercase Department",
  "type": "lower",
  "attributes": {
    "input": "attribute-1-value"
  }
}
```

| Field | Purpose |
|---|---|
| `name` | Human-readable identifier. Only used at the **root** level of a transform — nested/child transforms omit it. |
| `type` | The transform operation to run. This is the value documented as "Transform type" in each operation file (e.g. `lower`, `accountAttribute`, `concat`). |
| `attributes` | Configuration parameters for that operation. What's valid here varies per `type` — see each operation's doc. |

### Single vs. multiple inputs

Most transforms take one input and produce one output (e.g. `lower` turns `"Foo"` into `"foo"`). Some transforms accept multiple inputs — e.g. `concat` combines several strings (`"Foo"` + `"Bar"` → `"FooBar"`).

### Nesting / chaining transforms

A transform's `attributes.input` can itself be another transform object instead of a literal value, letting you chain operations — one transform's output becomes the next one's input. There is **no hard limit** on nesting depth, but the docs explicitly warn that deeply nested transforms become hard to maintain and read.

Example chain: a `replace` transform (swap `"Bar"` → `"Baz"`) feeds into a `concat` transform (join with `"Foo"`), which feeds into a `lower` transform — turning inputs `"Foo"`/`"Bar"` into final output `"foobaz"`.

### Implicit vs. explicit input

- **Implicit input**: if a transform's `attributes` has no `input` key, ISC supplies the input from context — e.g. whatever source attribute is mapped to this transform in an Identity Profile mapping.
- **Explicit input**: adding an `input` object inside `attributes` overrides the contextual value. This is commonly done with an `accountAttribute` transform to pull from a specific source rather than whatever the profile would otherwise use:

```json
{
  "name": "Lowercase Department",
  "type": "lower",
  "attributes": {
    "input": {
      "type": "accountAttribute",
      "attributes": {
        "attributeName": "department",
        "sourceName": "Source 2"
      }
    }
  }
}
```

This forces the `lower` transform to always read `department` from `Source 2`, regardless of which source the identity profile would otherwise pick.

### Template engine (Apache Velocity)

Transform attribute values can be strings containing Velocity template syntax (`$variableName`) that gets substituted at evaluation time. Available context variables differ by where the transform runs:

**Identity Attribute context** (running as part of an Identity Profile mapping):
- `identity` — the `Identity` object being processed
- `attributeDefinition` — the `ObjectAttribute` definition being populated
- `oldValue` — the attribute's previous value

**Account Profile context** (running as part of an account create/update provisioning policy):
- `field` — the account schema `Field` definition backing this attribute
- `identity` — the `Identity` being provisioned
- `application` — the `Application` (source) the account belongs to
- `current` — the attribute's current value on the account

### Where transforms run

| Context | Configured in | Behavior |
|---|---|---|
| **Identity Profile mappings** | Admin UI: **Identities** → **Identity Profiles** → **Mappings** | Runs during identity aggregation/refresh to compute identity attribute values. Test via the mapping UI's preview. |
| **Account Create/Update Profiles** (provisioning policies) | Admin UI: **Admin** → **Sources** → *(a source)* → **Accounts** → **Create Account**; or via REST provisioning-policy APIs | Runs only when a **provisioning event** fires (e.g. granting new access creates a new account) — **not** during aggregation. To test, trigger a real provisioning event (grant access to an identity with no existing account on that source) rather than expecting a preview against existing accounts. |

These two contexts are configured completely separately in the UI/API, even though the underlying transform JSON syntax is identical.

### Transforms vs. Rules

| | Transforms | Rules |
|---|---|---|
| Format | JSON | Code (typically BeanShell) |
| Editable by | Any org admin, via UI or REST API | Requires SailPoint review/installation |
| Preferred when | Whenever the needed logic is expressible in existing transform operations | Only when transforms genuinely can't do what's needed |
| Identity attribute equivalent | — | "Identity Attribute" rule |
| Account attribute (provisioning) equivalent | — | "Attribute Generator" rule |

**Recommendation from the docs:** default to transforms; drop to rules only for capabilities transforms don't cover.

### Limits and best practices

- **400KB** max file size per transform definition.
- Build and test small transform blocks incrementally before composing them into bigger nested chains.
- Use a real JSON-aware editor (VS Code, Sublime, Atom, etc.) to catch syntax errors and get formatting help — hand-editing raw JSON in a plain text box is error-prone.
- Reuse common patterns rather than re-deriving them — the `reference` operation exists specifically to let you write a transform once and reuse it by name elsewhere instead of copy-pasting the same nested structure repeatedly.
- There is often more than one valid way to build a given transform — prefer whatever is most readable/maintainable for your team over the "cleverest" option.
- Design defensively: account for source data that may be missing, malformed, or simply wrong (e.g. use `firstValid` to fall back when a field is null, rather than assuming every account has clean data).

---

## Promoting a transform between environments

When preparing a production change from a stage or sandbox transform, compare the complete JSON
definitions before writing the implementation plan. Do not copy the non-production definition
wholesale: source names, source IDs, Velocity branches, and unrelated fixes may differ between
environments.

Use the target environment explicitly for every command:

```bash
sail --env <lower-env> transform list
sail --env <lower-env> api get '/transforms/v1/<lower-transform-id>' --jsonpath '$' > /tmp/lower-transform.json
sail --env <production-env> transform list
sail --env <production-env> api get '/transforms/v1/<production-transform-id>' --jsonpath '$' > /tmp/production-transform.json
```

For a focused change:

1. Identify the transform by name in both environments and record both IDs.
2. Read both definitions and compare source names, account-attribute inputs, nested transform blocks, and the root `value`/Velocity expression.
3. Identify the exact intended delta and preserve production-only logic unless the change explicitly includes it.
4. Write the implementation plan in terms of the production definition: add the required block using production values, modify only the relevant branch, and leave unrelated branches unchanged.
5. Test representative identities in the lower environment, update the production transform through the dedicated transform command, and re-read the production definition to verify the saved JSON.

Treat a successful API status as insufficient evidence of a transform update. Verify the returned
definition, transform name/ID, changed JSON paths, and behavior with representative inputs. Preserve a
copy of the pre-change production JSON for rollback.

---

## CLI usage

`sail transform` is a dedicated subcommand — not `sail api` — for managing transform *definitions* (as opposed to the JSON structure/operations documented in this directory, which applies regardless of how you create/edit a transform).

> **Versioning note:** The current API specification defines the transforms resource at `/transforms/v1`. The dedicated `sail transform` subcommands abstract the underlying endpoint; use `sail transform --help` to inspect the installed CLI's options if an endpoint override is needed. The per-resource API version is also documented in [docs/api/README.md](../api/README.md#api-versioning).

| Command | Use it for |
|---|---|
| `sail transform list` | List all transforms in the tenant. |
| `sail transform preview -f <file> -i <identity> [-p <profile>] [-r]` | Evaluate a transform against live identity data without saving it — no `sail api` equivalent. |
| `sail transform download [-d <dir>]` | Bulk-export every transform to local files. |
| `sail transform create -f <file>` (or pipe JSON via stdin) | Create a transform from a JSON file. |
| `sail transform update -f <file>` | Update an existing transform from a JSON file. |
| `sail transform delete <id>` | Delete a transform by ID. |

Use these subcommands as the default path for transform management. Fall back to `sail api` only for what they don't cover (e.g. `GET` a single transform by ID with `--jsonpath` extraction) — see [SKILL.md](../../SKILL.md) for the general CLI-subcommand-vs-`sail api` split, which applies the same way to `sail workflow` and `sail rule`.

---

## Operation Index

Each row links to a dedicated file with the operation's exact `type` string, its `attributes`, worked examples, and any caveats.

| Operation | Type file | What it does |
|---|---|---|
| Account Attribute | [account-attribute.md](account-attribute.md) | Look up an account attribute for a particular source on an identity. |
| Base64 Decode | [base64-decode.md](base64-decode.md) | Render base64 data in its original binary format. |
| Base64 Encode | [base64-encode.md](base64-encode.md) | Encode data with a Base64-based text encoding scheme. |
| Concatenation | [concatenation.md](concatenation.md) | Join two or more string values into a combined output. |
| Conditional | [conditional.md](conditional.md) | Output different values depending on simple conditional logic. |
| Date Compare | [date-compare.md](date-compare.md) | Compare two dates and return a calculated value. |
| Date Format | [date-format.md](date-format.md) | Convert datetime strings from one format to another. |
| Date Math | [date-math.md](date-math.md) | Add, subtract, and round components of a timestamp's incoming value. |
| Decompose Diacritical Marks | [decompose-diacritical-marks.md](decompose-diacritical-marks.md) | Clean or standardize symbols used within language (e.g. accented characters). |
| Display Name | [display-name.md](display-name.md) | Use Preferred Name over Given Name to build an identity's Display Name. |
| E.164 Phone | [e164-phone.md](e164-phone.md) | Convert a phone number string into an E.164-compatible number. |
| First Valid | [first-valid.md](first-valid.md) | Return the first piece of data that is not null. |
| Generate Random String | [generate-random-string.md](generate-random-string.md) | Generate a random string of any length. |
| Get End of String | [get-end-of-string.md](get-end-of-string.md) | Get the rightmost N characters of a string. |
| Get Reference Identity Attribute | [get-reference-identity-attribute.md](get-reference-identity-attribute.md) | Get another user's identity attribute (e.g. a manager's). |
| Identity Attribute | [identity-attribute.md](identity-attribute.md) | Get a user's own identity attribute value. |
| Index Of | [index-of.md](index-of.md) | Get the location of a specific substring within a value. |
| ISO3166 | [iso-3166.md](iso-3166.md) | Convert a string into an ISO 3166 country code value. |
| Join | [join.md](join.md) | Join two or more string values into a combined output with a defined separator. |
| Last Index Of | [last-index-of.md](last-index-of.md) | Return the last location of a specific substring. |
| Left Pad | [left-pad.md](left-pad.md) | Left-pad an input string. |
| Lookup | [lookup.md](lookup.md) | Look up and return a key's matching value from a static table. |
| Lower | [lower.md](lower.md) | Convert an input string into all lowercase letters. |
| Name Normalizer | [name-normalizer.md](name-normalizer.md) | Clean or standardize the spelling of strings from source systems. |
| Random Alphanumeric | [random-alphanumeric.md](random-alphanumeric.md) | Generate a random alphanumeric string of any length. |
| Random Numeric | [random-numeric.md](random-numeric.md) | Generate a random number of any length. |
| Reference | [reference.md](reference.md) | Reuse a transform that has already been written elsewhere. |
| Replace All | [replace-all.md](replace-all.md) | Find and replace all instances of all matching patterns. |
| Replace | [replace.md](replace.md) | Find and replace all instances of a single string. |
| RFC5646 | [rfc-5646.md](rfc-5646.md) | Convert a three-letter language abbreviation to an RFC5646 language tag. |
| Right Pad | [right-pad.md](right-pad.md) | Add padding to the right of an incoming string. |
| Rule | [rule.md](rule.md) | Reuse rule logic that has already been written for a previous use case. |
| Split | [split.md](split.md) | Return the Nth element of a split array. |
| Static | [static.md](static.md) | Return a fixed string value. |
| Substring | [substring.md](substring.md) | Get the inner portion of a string. |
| Trim | [trim.md](trim.md) | Trim whitespace from both the beginning and end of an input string. |
| Upper | [upper.md](upper.md) | Convert an input string into all uppercase letters. |
| Username Generator | [username-generator.md](username-generator.md) | Derive a unique value for an attribute in an account create profile (e.g. a username). |
| UUID Generator | [uuid-generator.md](uuid-generator.md) | Create a universally unique ID (UUID). |

### Quick picks by task

- **Combining strings:** [concatenation.md](concatenation.md) (no separator control) vs [join.md](join.md) (defined separator)
- **Conditional/fallback logic:** [conditional.md](conditional.md) (if/then), [first-valid.md](first-valid.md) (first non-null)
- **Pulling data from elsewhere:** [account-attribute.md](account-attribute.md) (this identity, another source), [identity-attribute.md](identity-attribute.md) (this identity), [get-reference-identity-attribute.md](get-reference-identity-attribute.md) (another identity, e.g. manager)
- **String cleanup:** [lower.md](lower.md), [upper.md](upper.md), [trim.md](trim.md), [name-normalizer.md](name-normalizer.md), [decompose-diacritical-marks.md](decompose-diacritical-marks.md)
- **String slicing:** [substring.md](substring.md), [get-end-of-string.md](get-end-of-string.md), [split.md](split.md), [index-of.md](index-of.md), [last-index-of.md](last-index-of.md)
- **Generating values:** [static.md](static.md), [uuid-generator.md](uuid-generator.md), [random-numeric.md](random-numeric.md), [random-alphanumeric.md](random-alphanumeric.md), [generate-random-string.md](generate-random-string.md), [username-generator.md](username-generator.md)
- **Dates:** [date-format.md](date-format.md), [date-math.md](date-math.md), [date-compare.md](date-compare.md)
- **Encoding/standards:** [base64-encode.md](base64-encode.md), [base64-decode.md](base64-decode.md), [e164-phone.md](e164-phone.md), [iso-3166.md](iso-3166.md), [rfc-5646.md](rfc-5646.md)
- **Reuse:** [reference.md](reference.md) (reuse a transform), [rule.md](rule.md) (reuse a BeanShell rule)
