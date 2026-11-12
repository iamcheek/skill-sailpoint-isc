# Connector and customizer discovery → specification

Use when a user asks to start, continue, interview about, plan or specify a custom SaaS connector or customizer. Natural language is enough: “Start <application> connector discovery”, “Continue discovery for <task>”, “Build a spec from these decisions”, or “Plan a customizer for <source>”. Do not require a particular slash command. This is a technical workflow; use the user's issue-tracking skill or process (for example, Jira) for tracker operations and a source-control skill or process for requested branch/commit/PR/deployment operations when those are available. Never claim either ran just because it was named.

**Boundaries:** this workflow creates a design, not permission to create accounts, grant access, upload/link a connector, deploy or run disruptive tests. Offline research, interviewing and drafting require neither `sail` nor an ISC tenant. For live ISC discovery follow the parent skill's execution-path and environment selection; for all consequential writes and testing obtain the required explicit authorization. Protect secrets, raw configuration and unnecessary PII.

## 1. Start or resume

- Establish target system, desired outcome, connector **or** customizer, scope and users/owners where known. If an existing integration is named, load its [profile](../README.md) and inspect relevant project instructions and current repository/SDK evidence; use [configuration discovery](../configuration-discovery.md) only if live access is needed and authorized.
- Ask whether to use an existing discovery task or create one under the appropriate epic; the user's explicit request may supply this choice. Do not assume the project, epic, assignee, permissions or issue state. If the user requests issue-tracker operations and a suitable issue-tracking skill is available, follow its routine; it owns issue hierarchy, artifacts, comments and links. If unavailable, say so and offer an agreed local/restricted planning location; do not silently use a different channel or require a tracker for offline work.
- To resume from a task, read its current description, complete available comments and indexed current attachments through the team's issue-tracking skill or process. Resolve the artifact index and version history before deciding which is current; inspect the linked repository spec if one exists. If a required artifact is inaccessible, missing, truncated or contradictory, identify it and stop decisions that depend on it. You may continue unrelated, well-supported questions. Never invent the missing history.
- During discovery, the indexed Jira artifacts or agreed durable planning location are the working record. Don't keep the only copy in chat or a temporary directory when more sessions are expected. Respect issue access, attachment permissions and the repository's intended audience.

## 2. Research and interview in rounds

First search supplied requirements, relevant source/connector profiles, existing code/tests, SailPoint/SDK references and the target vendor's current API/permission documentation. Check current project/SDK versions and identify facts, inferred proposals and unknowns separately. Don't ask the user for a fact you can safely research, but don't access live tenants or private systems without the appropriate approval. Evidence should include a URL/path, version or revision, and date/environment when material.

Map decisions by prerequisite. Ask a manageable round of questions whose dependencies are already settled; give a recommended option and rationale when evidence permits. Wait for the user's answers before deciding or asking dependent questions. Ask for outcomes, scope, target populations/owners, identity and entitlement models, correlation, target permissions, supported operations, mappings, lifecycle, security, consistency, failure/retry and tests. For customizers also identify the underlying connector and exact hook, before/after data, preserved fields, additional calls, compatibility and rollback. Do not force every question into a small engagement.

Use concrete scenarios to resolve ambiguous terms and record a short vocabulary where helpful. For meaningful, difficult-to-reverse tradeoffs record alternatives and rationale; no separate ADR is mandatory. Preserve a structured question/answer record—not a verbatim conversation by default. Each decision needs a stable ID, status (`proposed`, `confirmed`, `blocked`, `superseded`), rationale, source, decision-maker if known and unresolved dependency. Recommendations are not user decisions. Offer a brief decision summary for confirmation; don't silently approve it.

## 3. Maintain artifacts and a decision map

Maintain: discovery brief/index, interview Q&A, decision log, research/source map, and proposed implementation outline. Use a map only for work complex enough to span sessions: destination, confirmed decisions (links to details), precise open questions and blockers, “not yet specified”, and out-of-scope. A research/decision item is not an implementation ticket. Keep unsupported ideas in the fog until they can be described precisely. Avoid large, duplicated transcripts and store sensitive evidence only in approved restricted locations.

When an issue tracker such as Jira is in scope, use the team's issue-tracking skill or process to attach or update the artifacts and the index, preserving prior versions. Confirm external writes when not already explicitly requested. Until publishing is confirmed, clearly label a draft as local/not uploaded. If Jira is unavailable, explain the limitation and ask whether to continue in an agreed durable local location; later reconcile with Jira rather than pretending it was updated.

## 4. Synthesize a repository design spec

When scope and enough decisions are clear, and an appropriate repository/location exists, draft the [implementation spec template](spec-template.md) from the evidence. The repository need not exist at the start; obtain approval for repository creation and applicable branch/commit operations. If feasibility fails, document the finding instead of forcing a repo or spec. Present the draft, key decisions, missing evidence, tests and per-slice readiness for user review. Don't quietly re-interview or fill missing inputs with guesses during synthesis.

The spec records requirements, design, SDK and target API versions, operation/hook contracts, mappings, risk, tests, decisions, and slices. It is not the SDK runtime `connector-spec.json`. A spec may be draft/partial: a slice is ready only when its own requirements, interfaces, dependencies, safety and verifiable tests are resolved; unrelated unknowns need not block it. A confirmed absence or unsupported feature must be distinguished from unavailable evidence.

Once reviewed and saved to the agreed repository location, link the path and revision from the task in the issue tracker. During discovery the indexed artifacts are the working record; after approval, the repository spec is the current implementation design and Jira retains the history. For later changes, update the design and record/link the decision in Jira, resolving conflicts explicitly. Do not claim an uncommitted file has a commit revision or a repository revision is deployed.

## 5. Slice and hand off

Propose small, independently verifiable behavioral slices with genuine blockers, spec links, acceptance criteria and test evidence. A shared prerequisite can be a separate slice when it cannot safely be vertical. Review granularity and dependencies with the user before any issue creation; use the team's issue-tracking skill or process for approved tasks and any existing connector UAT methodology. Avoid auto-labeling, assigning, closing or creating every question as a ticket. Discovery is complete only when its agreed criteria are met; no feasibility conclusion, spec, UAT or production approval may be fabricated.

For a subsequent session use [implementation/review](implementation-review.md): load the spec, current issue/artifact index, repository state and relevant references; flag version drift; work one approved ready slice; report tests, gaps and next pointer. Parallel subagents, worktrees, commits, PRs and live changes require applicable user/host authorization and are not assumed by this workflow.

## Related design inspiration

The decision-frontier interview, durable spec, behavioral slices and decision-map ideas were informed by [Matt Pocock's skills](https://github.com/mattpocock/skills/tree/4ae2317ddb22b046e26455e4f9c9b3aaaf62212d). This workflow is independently adapted for SailPoint, versioned source evidence and explicit safety/approval boundaries.
