# Connector-Specific Reference

Per-connector troubleshooting and configuration notes — one file per connector, loaded only when that specific connector is in play. This keeps connector-specific detail out of session context until it's actually needed, rather than baking it into a general investigation playbook that loads regardless of source.

| Connector | File | Covers |
|---|---|---|
| Salesforce SaaS | [salesforce.md](salesforce.md) | Vendor-documented known-issues pattern-match table — provisioning and authentication. |
| Web Services SaaS | [web-services-saas.md](web-services-saas.md) | General schema/mapping, parent-child operation, pagination, and aggregation guidance. |

Absence of a connector here doesn't mean nothing's known about it — it means no one's built its reference file yet. Add one when a connector-specific issue gets investigated and confirmed, following the existing files as a template:
- A short intro pointing back to this README.
- A pattern-match table: symptom/error text → cause → resolution → notes, clearly identifying vendor-documented entries.
- A "How to use this during an investigation" section describing when to check it and what to do if nothing matches.

## When to load a connector's file

From any investigation (provisioning, connection health, aggregation, etc.), once you've identified which connector/source is involved and have an actual error message or symptom in hand — not before. See the linking pattern in [../investigations/provisioning.md](../investigations/provisioning.md#connector-specific-troubleshooting) for how an investigation playbook should point here rather than embedding connector-specific content directly.
