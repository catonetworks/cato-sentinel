# Cato ASIM parsers for Microsoft Sentinel — preview

Deploy the preview parsers as workspace saved searches to normalize Cato events to Microsoft's Advanced Security Information Model (ASIM). Current breaking preview catalog **3.0.0** includes **6 schemas and 12 functions** (Microsoft schema version 1.0.0), aligned to Quick Review v32 and detailed mapping v47. Mapping remains a draft; this is not schema certification or a Content Hub release.

Choose the template for your table:

- **Existing custom table with a supported name:** [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel-ccf%2Fd96026ac193cd350ce4cdf3012237a026c3e1ab1%2Fasim%2FarmTemplate.json)
- **Standard `CatoNetworksEvents_CL` table:** [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel-ccf%2Fd96026ac193cd350ce4cdf3012237a026c3e1ab1%2Fasim%2FccfTemplate.json)

## Deploy in Azure portal

1. Select the subscription and resource group containing your existing Log Analytics workspace.
2. Enter `workspaceName`. The custom-table ARM template also asks for `tableName`; use the exact existing Cato custom-table name. The table must already exist and match ASCII `[A-Za-z][A-Za-z0-9_]*_CL` (first character an ASCII letter, remaining characters ASCII letters, digits, or underscores, ending `_CL`). Invalid names fail deployment; names are never sanitized. The CCF template takes only `workspaceName` and uses the fixed `CatoNetworksEvents_CL` table.
3. Review and create the deployment.

Both templates create workspace saved searches only. Your deploying identity needs normal ARM deployment permissions and `Microsoft.OperationalInsights/workspaces/savedSearches/write`; inspection and querying additionally require the corresponding read/query permissions. The CCF template uses aliases suffixed for `CatoNetworksEvents_CL`; the custom-table template uses aliases suffixed for your selected table. Both are built from the same canonical mappings.

## Deploy with Azure CLI

Set these shell variables to your own values; for the standard CCF template, set `TABLE_NAME` to `CatoNetworksEvents_CL` (the template itself has no table parameter):

```bash
SUBSCRIPTION_ID='<subscription-id>'
RESOURCE_GROUP='<resource-group>'
WORKSPACE_NAME='<workspace-name>'
TABLE_NAME='<existing-table-name>'
TEMPLATE_URI='https://raw.githubusercontent.com/catonetworks/cato-sentinel-ccf/d96026ac193cd350ce4cdf3012237a026c3e1ab1/asim/armTemplate.json'
```

Preview with what-if, then deploy incrementally:

```bash
az deployment group what-if --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME" tableName="$TABLE_NAME"
az deployment group create --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --mode Incremental --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME" tableName="$TABLE_NAME"
```

For `ccfTemplate.json`, set `TEMPLATE_URI` to its pinned raw URL and omit `tableName="$TABLE_NAME"` from both commands. Use **Incremental** mode only, never Complete.

## Query and integrate

Functions are saved searches, not ingestion changes. For a custom table named `MyCatoEvents_CL`, invoke a source function directly, for example:

```kusto
vimNetworkSessionCato_MyCatoEvents_CL(starttime=ago(1d))
```

For the standard table and CCF template:

```kusto
vimNetworkSessionCato_CatoNetworksEvents_CL(starttime=ago(1d))
```

To include Cato in a shared ASIM unifier, add its source-function invocation to the applicable unifier using the documented extension pattern. Merge without replacing existing sources; do not overwrite a built-in or other source's definition. A query can return zero rows when there are no matching events, the selected time range is empty, or fields/values do not match the parser. Zero rows alone do not indicate an installation failure.

Dashboards need events in the chosen table, required fields populated by those events, and a time range containing matching data. Empty results or panels can be valid when those conditions are not met.

## Catalog 3.0.0 mapping scope

| Schema | Cato EVENTS selected |
| --- | --- |
| NetworkSession | Security / Internet Firewall, WAN Firewall, LAN Firewall; IDS projection for IPS, Suspicious Activity, Dynamic Prevention, DNS Protection |
| Dns | Security with exact `DNS` array membership in `app_stack`; query and known response outcome required |
| WebSession | Security / Apps Security, or Security with exact `HTTP(S)` array membership in `app_stack`; usable URL required |
| Authentication | Connectivity / Authenticated |
| AlertEvent | Security / Anti Malware, SaaS Security API Anti Malware, SaaS Security API Data Protection, EPP Malware, Endpoint Protection, Data Security Alert, Data Discovery, Email Security Alert; **or any Detection and Response event**; **or System / Alert Triggered**. A usable `event_id` is required. |
| AuditEvent | System / Audit with operation and object |

Detection and Response and System / Alert Triggered are user-selected extensions, not Splunk-equivalent routes. Security alerts map to `Threat`; XDR Anomaly to `Anomaly`; XDR Account Operations/XDR Network and System / Alert Triggered to `Compliance Violation`; other Detection and Response subtypes to `Suspicious Activity`. `AlertName` uses scoped `indication`, then `rule_name`. `EventProduct` and cloud-fallback `Dvc` are `Cato Cloud`; `EventUid`, `EventOriginalUid`, and `AlertId` use `event_id` without an alert-ID fallback.

NetworkSession, including its IDS projection, uses `rule_name`; WebSession instead uses `network_rule`. `WANBOUND` maps to `Local`; `NO_DIRECTION` maps to `NA`. Translated-address scope is Internet/WAN Firewall, not LAN Firewall.

DNS emits only known response outcomes because ASIM outcome is mandatory; unknown/missing responses are not emitted as `NA`. `Domain` maps from `dest_domain` independently of `DnsQuery`; standard `domain_has_any` filtering still filters `DnsQuery`.

Removed NetworkSession projections include `SrcFQDN`, `DstHostname`, `DstDomainType`, `Hostname`, and `SessionId`. DNS removes `DnsResponseCodeName`; Authentication removes `TargetUsernameType`, `SrcDomain`, and `SrcDomainType`; WebSession removes `NetworkDuration`, `Duration`, and `HashType`. Unlisted normalized process/risk/MITRE roles are also retired. Raw source columns remain available. Standard filters on unmapped roles return no matches, never matches against unrelated raw fields.

Generic `file_hash` remains raw source context and cannot populate typed hashes or a generic alias without a verified algorithm; no explicit MD5 fallback is asserted. IPv4 is the baseline, with actual IPv6 support. PM-reviewed direct mappings of `src_endpoint_type`, `device_type`, `application_type`, and `full_path_url` are copied as requested and can fail ASIM logical-type checks. Companion username/device/application type fields are emitted only when their normalized parent identity exists. This draft is not schema-certified; Microsoft validation remains the release gate.

Epoch-millisecond source timestamps are already mapped to native `TimeGenerated` by the Cato connector's DCR. Parsers use retained original `time` when available, otherwise that mapped source clock, with `time_str` as an alternative when the primary clock is unavailable/invalid. Missing/null event counts become 1; malformed-present or invalid counts are excluded.

## Breaking upgrade from 2.0.x

Catalog 3.0.0 changes identifiers and mapped field roles; review dependent queries and customizations before upgrading. Incremental deployment updates the twelve active functions. If your workspace still has functions from the earlier eight-schema preview, it does **not delete** the four retired FileEvent/UserManagement saved searches. Back them up and review dependent queries/workbooks before removing them. Remove only their Cato source registrations from custom unifier unions, preserving all other providers, then explicitly remove retired aliases if no longer needed:

- `vimFileEventCato_<table>` and `ASimFileEventCato_<table>`
- `vimUserManagementCato_<table>` and `ASimUserManagementCato_<table>`

Replace `<table>` with the exact existing table name, including `_CL`. The installer never edits unifiers or deletes these functions; an old function remaining visible does not mean it is supported by catalog 3.0.0.

## Scope, permissions, and rollback

These templates deploy saved searches only. They do **not** change ingestion, the Data Collection Rule, RBAC, credentials, retention, or the connector, and do not recreate or modify the table. The root [`updateTemplate.json`](../updateTemplate.json) is separate: it updates the ingestion table schema and Data Collection Rule; it is not required to deploy these parsers.

Existing ARM/CCF connector customers can deploy these templates manually; no Content Hub update is needed. This installer does not enable automatic updates or change existing update settings. Back up any customized managed functions before deployment. Keep local forks outside Cato's aliases to avoid collisions. Roll back by redeploying the prior approved commit's template with Incremental mode. Incremental deployment does not delete newer functions absent from that template; remove them only after reviewing dependent queries and unifier registrations. A later Content Hub parser release must document migration from manual installations; this preview does not publish or validate that solution upgrade.

Parser upgrades are manual for existing ARM and CCF installations: deploy the approved version from this public repository when you choose to update. The daily ingestion-schema reconciliation does not update parser functions, and its auto-update setting does not opt a connector into parser updates. Cato's runtime application does not need new saved-search permissions for this manual workflow; the authorized customer/admin deployment identity performs the update.

Customer deployment requires no private GitHub access or Python. When an approved parser release is updated, both artifacts are regenerated from one canonical source and the pinned buttons are updated; do not hand-edit either template.
