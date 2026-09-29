# Salesforce SaaS Connector

Connector-specific reference for the **Salesforce SaaS connector** — see [docs/connectors/README.md](README.md) for how this catalog is organized and when to load a connector's file.

## Known issues

This is a **pattern-match reference**, not a procedure. Once you have an actual error message or symptom from connector logs, account-activity errors, or a source-health check, scan it against the table below before re-deriving the root cause. If nothing matches, fall back to the general investigation methodology in [provisioning.md](../investigations/provisioning.md).

The entries below are based on SailPoint's vendor troubleshooting documentation:
https://documentation.sailpoint.com/connectors/saas/salesforce/help/saas_connectivity/salesforce/troubleshooting.html

| # | Match on (exact error text / symptom) | Cause | Resolution | Notes |
|---|---|---|---|---|
| 1 | `OpenError: Cannot set or reset the password for a single sign-on user.` | In SSO-enabled Salesforce orgs, the connector's composite create-account call bundles an automatic password-reset sub-request. Salesforce rejects password resets for SSO users, and because the composite transaction is atomic, the whole account-create rolls back with it. | In the source's **Aggregation Settings** tab, enable **Disable User Creation Email Notification** so the connector does not issue the password-reset/notification sub-request for new accounts. | Applies to SSO-enabled Salesforce orgs. |
| 2 | `INSUFFICIENT_ACCESS: portal user email settings are not available` | Partner/community users are not created with the default configuration when **Disable User Creation Email Notification** is unchecked. | In Salesforce, activate the community (**Setup > All Sites > Workspaces > Administration > Setting** → Active), then select the user's profile under **Setup > All Sites > Workspaces > Administration > Members > Profiles** and save. | Applies to Salesforce partner/community users, not standard internal users. |
| 3 | `DUPLICATE_VALUE` or a "Duplicate User" error on account create, where the email is the conflict | The email address is already in use elsewhere in the Salesforce org. Email is the mandatory username field for new Salesforce users. | Create the account with a different email, or resolve the existing duplicate in Salesforce before retrying. | Distinct from duplicate permission-set-assignment errors; inspect which Salesforce object is named. |
| 4 | Connector fails to create a new account when Salesforce's **total active account limit** is exceeded | Salesforce orgs have a cap on total active user accounts, based on licensing. | Increase the account limit, if licensing permits, or deactivate unused active accounts before retrying. | Distinct from exhaustion of a specific permission-set or feature license. |
| 5 | `INVALID_SESSION_ID` during a Salesforce connector operation | Stale or invalid cached session data on the source configuration. | Remove the `serverUrl` and `sessionID` attributes from the source config via the source update API, run a test connection, and retry the operation if the test succeeds. | Check the source configuration before making broader changes. |
| 6 | Errors while setting up **JWT-flow authentication** (self-signed certificate, keystore, or connected-app configuration) | JWT setup spans certificate creation, keystore conversion, connected-app configuration, and key validation. | Follow Salesforce's JWT authentication setup procedure. Use `keytool`/`openssl` carefully when converting `.jks`, `.p12`, `.pem`, and `.key` files. | Relevant only when the source uses JWT authentication. |
| 7 | `Unable to Authorize User: user is not admin approved to access this app` during a JWT test connection | The connected app's OAuth policy has not pre-authorized the service account or subject user. | In the connected app, use **Manage > Edit Policies**, select **Admin approved users are pre-authorized for Permitted Users**, then authorize the relevant profiles or permission sets. | JWT-specific. |
| 8 | `PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target` | Certificate trust-path issue during the connector-to-Salesforce TLS handshake. | Verify the service-account profile permissions and that the OAuth token URL points at the intended production or sandbox org. | Can occur independently of JWT setup. |

## How to use this during an investigation

1. Once you have an actual error message or symptom, scan it against the **Match on** column.
2. Treat the listed resolution as a starting point and verify it against the specific source configuration.
3. If nothing matches, use the general provisioning or connector investigation methodology rather than assuming the table is exhaustive.
4. Add future vendor-documented Salesforce patterns here rather than starting a parallel list.
