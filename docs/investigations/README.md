# Investigation Playbooks

Task-oriented playbooks for specific kinds of investigations in ISC, as opposed to pure API/CLI reference (see [docs/api/](../api/README.md), [docs/cli/](../cli/README.md), [docs/search/](../search/README.md)). Load one of these when the user's request matches its trigger condition.

| Playbook | Load when |
|---|---|
| [access-request-lookup.md](access-request-lookup.md) | User provides an external ticket number (ServiceNow RITM/REQ, or similar) and asks what happened to the access request behind it — no identity/source/failure known yet, just a ticket ID. |
| [provisioning.md](provisioning.md) | User asks to investigate provisioning failures or activity for an identity (e.g. "why didn't this access request provision", "what happened when I granted X access to Y") — identity/source already known, or resolvable by name. |

Once a playbook identifies which connector/source is involved, also check [docs/connectors/](../connectors/README.md) for a per-connector troubleshooting reference — see provisioning.md's [Connector-specific troubleshooting](provisioning.md#connector-specific-troubleshooting) section for the pattern.

Named "investigations" rather than "workflows" because ISC has its own distinct "Workflow" feature (with its own `WORKFLOW_*` audit events, etc.) — these files are playbooks *for us to follow*, not documentation *about* that feature.
