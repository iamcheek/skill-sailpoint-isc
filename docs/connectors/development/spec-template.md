# Implementation design spec template

Copy or adapt this **design** template into the integration repository, for example `docs/specs/<integration>-connector.md`, once there is an agreed location. It is not SailPoint's runtime `connector-spec.json`, a deployment record or proof of testing. Do not require empty sections unrelated to the project; explicitly mark consequential missing facts **unknown**, and mark out-of-scope areas **not applicable** with a reason. Link evidence rather than embedding credentials, PII or raw API/configuration dumps.

```markdown
# <Integration> — <connector | customizer> implementation spec

**Status:** Draft | Reviewed | Approved | Superseded
**Owner/reviewer:** <name or unknown>  **Reviewed:** <date>
**Discovery:** <issue or durable record>  **Repository/branch/revision:** <or unknown>
**SDK version / evidence:** <resolved version, lockfile/source revision or unknown>
**Target APIs / versions / evidence:** <URLs and revisions or unknown>
**Environments / source:** <scope or unknown; no secrets>

## Outcome, scope and exclusions
<Business outcome, affected populations/access, accepted requirements, exclusions and source links. Label proposed vs approved.>

## Architecture and domain vocabulary
<Systems and boundaries; data flows; terminology that could be confused; configuration and auth approach, required permissions, access boundaries.>

## Identity, schema and mappings
<Target identifiers, stable lookup/unique keys, ISC identity/correlation vs target joins; account and each entitlement type/association; table of input → transform → output, type, required/null/multivalue behavior, source and acceptance criteria. Mark not applicable for customizer only with reason.>

## Operation contracts
| Requirement ID | Command or customizer hook | Input/output and mapped target API | Behavior / negative and partial-failure cases | Evidence / status |
|---|---|---|---|---|
| <R1> | <verified command/hook> | <contract or unknown> | <expected behavior> | <link + approved/proposed/unknown> |

<For customizers: underlying connector/version, exact before/after/named operation, untouched fields, config, execution assumptions, dependencies, failure effect, compatibility and unlink/rollback. For connectors: runtime spec declarations, schema/policy assumptions, handlers and target endpoint contracts.>

## Reliability, security and operational decisions
<Pagination, rate limits, target consistency, state/checkpoints, retries/idempotency, partial writes and recovery, timeout/size constraints, logs/redaction, credential lifecycle and safety approvals. Cite sources; avoid asserting an unverified behavior.>

## Decision and evidence index
| ID | Decision / rationale / alternatives | Status and decision-maker | Evidence / version / date |
|---|---|---|---|
| <D1> | <decision> | proposed / confirmed / blocked / superseded | <link> |

## Verification plan
| Requirement or risk | Test seam and scenario | Expected result | Evidence and status |
|---|---|---|---|
| <R1> | <mocked contract, local handler or controlled live/UAT> | <observable outcome> | <not run / result link> |

<Explicitly separate offline unit/contract, local connector, approved live, UAT and stakeholder acceptance. Capture safe test population, environment, cleanup and evidence policy when live testing is planned.>

## Implementation slices and dependencies
| Slice | Deliverable / independently verifiable behavior | Depends on | Ready? | Tests / acceptance |
|---|---|---|---|---|
| <S1> | <outcome> | <none or IDs> | ready / blocked: reason | <observable check> |

## Open questions, risks and handoff
| ID | Unknown / conflict | Blocking which slice? | Decision owner / next evidence |
|---|---|---|---|
| <Q1> | <question> | <slice or none> | <owner/next step> |

**Next session:** <one approved ready slice, relevant artifacts and exact next action>
**Changes since review:** <date, decision, link or none>
```

## Readiness check

For each slice confirm: approved scope and observable acceptance; SDK/hook/target API input-output contract and relevant versions; identifiers and mapping; authorization/secret handling; error, partial-write and retry behavior; required mock tests and any live safety gate; and no unresolved dependency affecting that slice. Mark blocked with the specific missing fact if not. The spec can remain draft while some slices are ready; user approval of a slice, repository write, Jira publication and live operation are separate decisions.
