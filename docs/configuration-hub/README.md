# SailPoint Configuration Hub

Use Configuration Hub when planning or executing migration, backup, comparison, and deployment of supported Identity Security Cloud configuration objects between tenants. The target tenant is the live tenant, commonly production; the source tenant is the connected tenant from which configuration is backed up.

This reference is based on SailPoint's Configuration Hub documentation:

- [Using the Configuration Hub](https://documentation.sailpoint.com/saas/help/confighub/index.html)
- [Working with Backups](https://documentation.sailpoint.com/saas/help/confighub/config_hub_backup.html)
- [Working with Configuration Files](https://documentation.sailpoint.com/saas/help/confighub/config_hub_config_file.html)
- [Reviewing Deployment Activity](https://documentation.sailpoint.com/saas/help/confighub/config_hub_activity.html)
- [Using Tenant Connections](https://documentation.sailpoint.com/saas/help/confighub/config_hub_ten_conn.html)
- [Viewing Scheduled Jobs](https://documentation.sailpoint.com/saas/help/confighub/scheduling.html)
- [Mapping Objects](https://documentation.sailpoint.com/saas/help/confighub/config_hub_mapping.html)
- [Configuration Hub Cloud Storage and Advanced Settings](https://documentation.sailpoint.com/saas/help/confighub/config_hub_cloud_storage.html)
- [SaaS Configuration supported objects](https://developer.sailpoint.com/docs/extensibility/configuration-management/saas-configuration/)

## What Configuration Hub does and does not do

Configuration Hub backs up, compares, restores, and deploys supported configuration objects. It is not a full system backup. The current SaaS Configuration supported-object list identifies **Form Definitions** (`FORM_DEFINITION`) and **Workflows** (`WORKFLOW`) as supported for Configuration Hub backup and deploy, which makes Configuration Hub an appropriate deployment mechanism for forms and workflows. Confirm any dependent object types separately before deploying; the authoritative supported-object list is maintained in SailPoint's [SaaS Configuration documentation](https://developer.sailpoint.com/docs/extensibility/configuration-management/saas-configuration/).

Configuration Hub does not back up secrets or credentials. Source credentials and connection values may need to be re-entered or manually reconnected after deployment. A deployment also does not automatically delete objects that are present in the live tenant but absent from the backup; those objects appear in the `Not in Backup` reference list and require deliberate manual handling.

Use least-privilege Configuration Hub access:

- **Reader** can view Configuration Hub content but cannot make changes.
- **Backup Administrator** can work with backups and has read access to tenant connections, but cannot create tenant connections.
- **Configuration Hub Administrator** is required for full administration, including cloud storage and feature enablement.

## Tenant connections and environment safety

Before migrating configuration:

1. Identify the source tenant and the target/live tenant explicitly.
2. Confirm both tenants are in the same region and that network settings and Identity Profiles permit the connection.
3. When using a tenant connection, create a Personal Access Token in the connected/source tenant with the `sp:config:backup-connection` scope.
4. Add the connection while logged in to the target tenant. Connections are per user and are not shared.
5. Confirm the target/live tenant using the green live-tenant indicator before creating a draft or deploying.

A tenant can have up to 10 tenant connections. Never infer the target from a tenant name alone when production changes are involved.

## Controlled deployment workflow

Use this sequence for a production change such as deploying a Form and Workflow.

### 1. Preflight

- Confirm the approved change window, source tenant, target production tenant, implementer, and approver.
- Confirm that the Form is represented as a `FORM_DEFINITION` and the workflow as a `WORKFLOW`; both are supported for Configuration Hub backup and deploy. Confirm any dependent object types separately. If a required dependency is unsupported or does not appear in the draft, stop and use the approved alternative deployment method rather than assuming it will be migrated.
- Review the source configuration for tenant-specific values, references, secrets, credentials, and dependent objects.
- Verify the operator has the required Configuration Hub role and that tenant connections and network access are healthy.

### 2. Create a named source backup

Create a manual backup in the source tenant with a name that includes the change identifier and purpose, for example `<change-ID> production deployment source backup`.

Prefer selecting only the required object types and exact object names. Object names entered in `Objects by Name` are case-sensitive. Leaving the name field blank while selecting an object type includes all objects of that type, so avoid an unnecessarily broad backup.

Review the backup summary and object details. Capture the object counts and JSON details as pre-deployment evidence. Manual backups are limited to 10 per tenant; automated production backups are retained by SailPoint at up to five weekly backups. Backups and drafts normally support up to 5,000 objects, while draft creation can support up to 30,000 objects and deployment is limited to 5,000 unless SailPoint enables a higher limit.

### 3. Create the target draft

From the target/live production tenant, select the source backup and create a clearly named draft, such as `<change-ID> production deployment draft`.

Treat the draft summary as the deployment plan. Review each category:

- **Adds to Live** — objects that will be added.
- **Modifies to Live** — live objects that will be changed.
- **Not in Backup** — live objects absent from the source backup; these are not automatically deleted.
- **Reference Issues** — unresolved dependencies or references that may prevent a successful deployment.

Resolve reference issues before deployment. Configuration Hub uses object type and name to substitute referenced IDs where possible, but an object containing only an ID may not be resolvable until deployment. Use Object Mapping when source and target values differ.

Edit the draft to remove unrelated objects and adjust supported object JSON only when necessary. Object IDs cannot be edited or removed. Saving draft changes permanently removes objects removed from that draft; if the wrong objects are removed, create a new draft from the backup. The source backup itself is not changed.

### 4. Apply mappings and approval controls

Object mappings are configured under **Advanced Settings > Object Mapping**. A mapping requires:

- Object type
- JSON Path beginning with `$`, such as `$.owner.name`
- Original/source value
- New/target value
- Enabled status

New mappings are disabled by default. Enable only mappings intended for this deployment, and verify that the mapping does not substitute a tenant-specific owner, source, endpoint, or other value incorrectly. Configuration Hub also applies global substitution rules for tenant-specific values, defaults, ignored comparison attributes, and array ordering; review the resulting draft rather than assuming every substitution is correct.

If Configuration Hub approvals are enabled, request approval from the draft summary only after the draft is complete. A draft cannot be edited while an approval request is pending. Configuration Hub approval does not replace the organization's ServiceNow change approval.

### 5. Deploy in the approved window

Before deployment, perform a final review of the draft objects, reference issues, mappings, object counts, and target-tenant indicator. Deploy immediately or schedule the deployment for the approved window. If scheduling is used, verify the schedule's time zone and action type; Configuration Hub permits at most one scheduled action for each of `BACKUP`, `DRAFT`, and `DEPLOY`. A non-recurring draft or deployment action is deleted after it triggers.

For a Form and Workflow change, deploy only the approved objects and their required dependencies. Do not treat a successful Configuration Hub deployment as proof that the new workflow has been functionally validated.

### 6. Validate and retain evidence

After deployment:

1. Review the Configuration Hub Activity Log.
2. Confirm the result is **Complete**, not **Partially Complete** or **Failed**.
3. Open the deployment event and retain the Deployment Log and Deployment Draft details.
4. Search audit events for `Update Config Passed` or `Update Config Failed` as an independent audit cross-check.
5. Validate the deployed Form and Workflow with the change-specific smoke test, including the expected target population and behavior.
6. Document warnings, failed objects, manual reconnections, and evidence locations in the change record.

A **Partially Complete** result requires investigation of the failed objects before the change can be considered successful.

## Backout considerations

A Configuration Hub rollback is a new controlled deployment, not an automatic undo. Use the pre-change backup to prepare a new draft, review the resulting changes, obtain required approval, and deploy it to the target tenant. Disabling the new Form or Workflow may be an immediate containment step, but it does not reverse configuration changes already deployed or identity processing already completed.

After a restore or deployment, verify any manual reconnections required by the affected objects. SailPoint specifically calls out password policies that must be reconnected to sources and service desk integrations that must be reconnected to virtual appliances. Secrets and credentials are not restored by Configuration Hub.

## Cloud storage and artifact retention

Cloud Storage can automatically transfer backups and deployment artifacts to an AWS S3 bucket. The bucket must be in the same AWS region as the SailPoint tenant and must grant SailPoint `s3:PutObject` permission. Configuration Hub cannot read or delete artifacts in the bucket. A connection test writes `sailpoint-config-hub-transfer-test.json` to the bucket root.

When enabled, artifacts use paths such as:

```text
/sp-config-backups/<tenant>
/sp-config-deploy/<tenant>
/sp-config-deploy/<tenant>/historicalDrafts
```

Successful deployments transfer both the deployment log and the draft configuration file. Artifacts created before cloud storage was enabled are not automatically transferred; use the Sync action if historical artifacts need to be copied.

## Operational limits and cautions

- Manual backups: up to 10 retained per tenant.
- Automated production backups: weekly, up to five retained by SailPoint.
- Configuration files uploaded: up to 10.
- Drafts: up to 10; creating a new draft can delete the oldest undeployed draft at the retention limit.
- Draft creation: up to 30,000 objects by default.
- Draft deployment: up to 5,000 objects by default.
- Tenant connections: up to 10, same-region requirement.
- Configuration Hub does not back up secrets or credentials.
- Deployment does not delete objects listed under `Not in Backup`.
- Object IDs cannot be edited or removed from a draft.
- New mappings are disabled until explicitly enabled.
- A partial deployment must be treated as an incident requiring review, not as a successful deployment.
