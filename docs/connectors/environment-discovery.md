# Environment connector discovery → draft profiles

Use when asked to inventory the connectors in an ISC tenant, find the SailPoint documentation for each source, or create or refresh integration profiles for a whole environment. The routine lists the tenant's sources, matches each one to SailPoint's connector documentation, and drafts a profile per integration. Those profiles then hold the documentation pointers, dated configuration facts and the known-issues catalog that troubleshooting sessions add to.

It works with [profile-building](profile-building.md), which defines the profile location, structure and update rules, and with per-source [configuration discovery](configuration-discovery.md), which supplies deeper facts. It does not replace either.

**Boundaries:** ISC access is read-only. The only writes are local profile files, made after the user approves them. Follow the parent skill's execution-path and environment selection before the first call. Never print or store connector configuration, credentials or raw source responses; keep only the projected fields below.

## 1. Resolve the profile folder

1. Identify the tenant **subdomain** from the selected environment's base URL (`sail environment list` shows `baseurl`; `iscUtils` shows `Tenant`). Use the subdomain, not the `sail` environment alias, as the tenant key; aliases are personal and differ between users.
2. Resolve the profile root and organization folder as described in [profile-building → Profile location](profile-building.md#profile-location). If the tenant is not listed in any organization's `tenants.md`, ask which organization it belongs to (or whether to create one) before writing anything.
3. Read only that organization's folder. Do not open, compare against or copy from another organization's profiles.

## 2. Inventory sources (read-only)

Page through every source and keep only identifying fields. Connector configuration is never projected:

```bash
ISC_ENV='<selected-environment>'
: > /tmp/isc-sources.jsonl
offset=0
while :; do
  page=$(sail --env "$ISC_ENV" api get /sources/v1 --query "limit=250" --query "offset=$offset" \
      --query "sorters=name" --jsonpath '$' 2>/dev/null | tail -1 \
    | jq -c '.[] | {id, name, type, connectorName, connector,
                   cluster: (if .cluster then .cluster.name else null end), authoritative}')
  [ -z "$page" ] && break
  echo "$page" >> /tmp/isc-sources.jsonl
  n=$(echo "$page" | wc -l); offset=$((offset + n)); [ "$n" -lt 250 ] && break
done
```

In the PowerShell path, `Get-ISCSource -List` (`/v3/sources`) returns the same objects; select the same properties and discard the rest. Delete the temporary file when the routine is finished.

## 3. Load the SailPoint connector catalog

Fetch `https://documentation.sailpoint.com/connectors/help/llms-full.txt` once per run. It is the whole connector catalog in a single file (about 100 KB): the category pages (Deep Governance, Quick Compliance, Discovery, Service Desk, Single Source, Credential Providers and so on) joined together. Do not load it into context. Parse it with a script:

```python
import re, sys
rows, category, name = [], None, None
for line in open(sys.argv[1]):
    if m := re.match(r'^# (.+)', line):
        category = m.group(1).strip()
    elif m := re.match(r'^- \*\*(.+?)\*\*', line):
        name = m.group(1).strip()
    elif name and (m := re.match(r'^\s+- \[(.+?)\]\(<(https://documentation\.sailpoint\.com/connectors/[^>]+/help/)index\.html>\)', line)):
        rows.append({"category": category, "name": name, "guide": m.group(1),
                     "index": m.group(2) + "index.html", "llms": m.group(2) + "llms.txt"})
```

Each row is a catalog name, its guide label (`SaaS-Based Connector Guide`, `VA-Based Connector Guide` or another label) and its URLs. **The catalog links point to `index.html`; replace that with `llms.txt` to get the connector's agent-readable page index.** Verified 2026-11-12: all 594 guide links in the catalog resolved this way, and each returned a distinct index with markdown page links. The parser above captures the bolded-name entries (572 at that date); credential-provider entries use a different layout and are rarely ISC account sources.

## 4. Match each source to a guide

Apply these rules in order and record a confidence for every source:

1. **Match on `connectorName`, not `type`.** For SaaS connectors, `type` is a connector UUID; `connectorName` carries the display name (`Salesforce SaaS`, `SCIM 2.0 SaaS`, `Web Services SaaS`).
2. **Choose SaaS vs. VA from the cluster, not the name.** A cluster of `sp_connect_proxy_cluster` means SaaS; a named VA cluster means VA. Most connectors have separate SaaS and VA guides with different settings and limits, so the wrong one gives wrong advice.
3. **No cluster means unresolved.** Use secondary evidence. In one tenant the classic `delimited-file-angularsc` connector (name `Generic`) matched the original Delimited File guide, while the SaaS variant is titled "Delimited File SaaS"; `-angularsc` connector IDs were also seen on VA-based Okta and Workday sources. Treat this as an observed hint, not a rule, and ask the user to confirm.
4. **Strip a trailing ` SaaS` and compare names.** If the names differ beyond that, mark the match as an **alias** and check the guide's `index.md` title. Examples seen: `ServiceNow SaaS` → "ServiceNow Identity Governance Connector"; `Workday Accounts SaaS` → "Workday Accounts", which is a different guide from "Workday".
5. **Custom connectors have no vendor guide.** A connector built with the SaaS connector SDK and uploaded to the tenant will not match. Mark it **custom** and point to its repository, [development guide](development/README.md) and profile.
6. **Several sources can share one guide.** Generic connectors (SCIM 2.0, Web Services, Delimited File, JDBC) serve many target applications. The guide describes the connector, not the application behind it.

Confidence values: **exact**, **alias** (confirmed from the guide title), **inferred** (needs confirmation) or **custom**. Present the result for the user to validate before drafting anything:

| Source | `connectorName` / `connector` | Runs on | Guide (`index.html`) | Match |
|---|---|---|---|---|

## 5. Group sources into integrations

Profiles are per **integration**: the governed application and its business purpose, across every environment it is connected in. A profile is not per source or per connector. Propose a grouping and ask the user to confirm it. For example, `<App> Prod`, `<App> Pre-Prod` and `<App> Sandbox` sources may be one integration with three target environments. Two Web Services sources pointing at different applications are two integrations that share one connector guide. Record the matching source in each tenant listed in `tenants.md`.

## 6. Draft or refresh each profile's connector documentation

For each integration:

1. Fetch the matched connector's `llms.txt` (usually 3–28 pages). Select the pages relevant to this integration: connecting/authentication, schema, provisioning, supported features and troubleshooting. Fetch those `.md` pages only. Use the connector's `llms-full.txt` (roughly 12–27k tokens) only for a deliberate full review. Treat any response other than HTTP 200 with `text/markdown` as missing: the site returns `404 text/html` for pages that do not exist (verified 2026-11-12).
2. Write the profile's **Connector documentation** section as described in [profile-building](profile-building.md#profile-structure): the guide and `llms.txt` links, SaaS/VA, the ISC `connectorName`/`connector` values, the reviewed date and a short **read before acting** list. If the connector is generic (Web Services, SCIM, JDBC, Delimited File), also link the matching [generic reference](generic/README.md) when one exists. Do not copy vendor content into the profile. Keep links and short interpretive notes, and fetch pages when a task needs them.
3. **Flag, do not repeat, unsafe or conflicting vendor guidance.** Examples: steps that send private keys or secrets to a third-party site; advice that conflicts across pages of the same guide; defaults with large effects (provisioning disabled by default, ID changes after aggregation creating duplicates). Label each as a vendor-documentation observation with its page link.
4. **New integration:** create a draft profile with the identification, connector documentation and source-per-tenant sections filled in. Mark requirements, design rationale and owner **unknown**. Do not invent them. Run [configuration discovery](configuration-discovery.md) only if the user asks for the deeper inventory.
5. **Existing profile:** never overwrite it. Propose a diff limited to the connector documentation and source-per-tenant sections, and keep prior dated observations.
6. Show the proposed new files and diffs, and write them only after approval. Add new profiles to the organization's index (its `README.md`, or the connector index when the profiles live inside the skill).

## 7. Report

Return the validated match table, the integration grouping, the profiles created or proposed for update, any unresolved or custom matches, and the catalog and guide fetch date. A catalog match proves which documentation applies to a source. It does not show that a feature is configured, licensed or working.
