# Implement and review a connector spec

Use for a requested implementation slice or a connector/customizer review. Start with [discovery](discovery.md), the [SDK reference](sdk.md) and a reviewed repository spec when available. A design spec is not the runtime `connector-spec.json` and neither proves deployment.

## Resume and select work

1. Read the agreed spec, discovery index and decision records, relevant issue and latest comments if available, repository instructions, branch/working tree and the exact SDK lockfile/version. Load only relevant [commands](commands.md), [customizer](customizers.md), [runtime](runtime-testing.md) and integration profile. If an artifact cannot be accessed, report it and block only dependent work; do not silently infer its contents.
2. Compare approved decisions with current code, target API documentation and installed SDK. Label and surface discrepancies; agree a resolution before implementing affected behavior. Verify no proposed slice changes scope or relies on an unapproved assumption. Select one approved slice whose real blockers are resolved; do not make the agent-ready label or a ticket status stand in for that check.
3. Inspect existing tests and seams, then implement the approved behavior using the target's actual API contract. Choose meaningful test seams that exercise externally visible behavior; don't force one seam across unrelated target operations. Write/run tests supported by the project. Keep changes to the slice and preserve existing work. If the user authorizes parallel agents, isolate writers and account for shared files and integration conflicts; do not delegate or create branches merely because a graph exists.
4. Run applicable typecheck, unit/contract tests, lint and packaging checks after inspecting scripts and side effects. Record exact commands, result and what was not run. Ask separately before credentialed target tests, source changes, provisioning, upload/link, deployment or other consequential actions; apply the parent safety overrides.
5. Update the design/handoff or Jira only as requested/authorized: implementation evidence, commit or working-tree state, tests, unresolved decisions and next slice. Do not close or transition a task because code or a mock test passed. For issue-tracker changes, commits, PRs or deployment, use the team's issue-tracking and source-control skills or processes when available.

## Review independently against a fixed scope

Identify the review base and include the requested committed changes plus relevant staged, unstaged and untracked files; a three-dot `HEAD` diff alone omits working-tree changes. Confirm the spec/version and actual review scope. Report separate, cited findings by:

- **Spec and acceptance:** missing or incorrect behavior, unauthorized scope, unresolved assumptions and per-criterion evidence.
- **Repository and SDK/target contracts:** repo instructions, correct versioned types/commands/hooks, key and mapping consistency, schema/policy relationships, pagination, state, retry/partial failures, credential protection and destructive or privilege-impacting paths.
- **Tests and evidence:** missing negative/edge cases, mock vs live results, unverified deployed version and safety/rollback gaps.

Cite file and line, requirement or source, impact and actionable remedy. Confirm findings against the actual code and evidence; distinguish a defect from a concern and a test gap from a proven failure. If a spec or standards source is unavailable, state the limit rather than inventing it. An independent review is not proof of live behavior or acceptance; do not run an unbounded review/fix loop. For an actual PR, follow any separate pre-PR review gate the team uses rather than claiming this checklist satisfies it.
