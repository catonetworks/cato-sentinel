# Cato ASIM parsers for Microsoft Sentinel — preview

Parser version **3.0.1** provides six ASIM schemas and 12 query-time functions, targeting Microsoft schema version 1.0.0: `NetworkSession`, `Dns`, `WebSession`, `Authentication`, `AlertEvent`, and `AuditEvent`. The parsers normalize Cato events already present in a Log Analytics table. Installation creates saved searches, not new tables or ingestion configuration. Version 3.0.1 improves Microsoft Sentinel compatibility without changing event classification or field mappings.

This is a preview, not a Microsoft-certified schema or a Content Hub release. Entity-type values and URLs in `DstFQDN` may not satisfy ASIM logical types. Missing source fields can produce sparse or empty normalized output. Generic `file_hash` does not identify a verified hashing algorithm, so it is not assigned a typed hash.

## Choose the parser template

Use **CCF** for the Content Hub's Cato Networks Events (Push) connector, or **ARM-template connector** for the Cato integration with a customer-selected table. Both installer files use the ARM template format.

| Connector flavor | Template | Table | Parameters |
|---|---|---|---|
| CCF | [![Deploy CCF parsers](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel%2Fa26a985e510b0472d974cceb3e747df2e21ac75d%2Fasim%2FccfTemplate.json) | Fixed `CatoNetworksEvents_CL` | `workspaceName` |
| ARM-template connector | [![Deploy ARM parsers](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel%2Fa26a985e510b0472d974cceb3e747df2e21ac75d%2Fasim%2FarmTemplate.json) | Customer-selected existing Cato table | `workspaceName`, `tableName` |

The selected table must already exist. ARM `tableName` must match ASCII `[A-Za-z][A-Za-z0-9_]*_CL`: start with an ASCII letter, continue with ASCII letters, digits, or underscores, and end in `_CL`. Names are not rewritten. Both templates use the same mappings; table-specific function aliases are created for the selected table.

Your deployment identity needs normal ARM deployment permissions and `Microsoft.OperationalInsights/workspaces/savedSearches/write`. Reading and querying saved searches require the corresponding permissions. Deploy into the subscription and resource group containing the existing Log Analytics workspace.

## Deploy in Azure portal

**CCF:** Open the CCF deployment link above, select the subscription and resource group for the workspace, enter `workspaceName`, then review and create. The table is fixed; there is no `tableName` parameter.

**ARM-template connector:** Open the ARM deployment link above, select the subscription and resource group, enter `workspaceName` and the exact name of the existing Cato table as `tableName`, then review and create.

Use **Incremental** mode only, never Complete mode. These parser workflows leave the table schema and DCR unchanged. For CCF schema updates, use the separate CCF-only [`updateTemplate.json`](../updateTemplate.json) guide in the [repository README](../README.md); it does not apply to ARM connector schema updates.

## Deploy with Azure CLI

Set the deployment values first:

```bash
SUBSCRIPTION_ID='<subscription-id>'
RESOURCE_GROUP='<resource-group>'
WORKSPACE_NAME='<workspace-name>'
```

### CCF connector

Run the CCF template without a table-name parameter:

```bash
TEMPLATE_URI='https://raw.githubusercontent.com/catonetworks/cato-sentinel/a26a985e510b0472d974cceb3e747df2e21ac75d/asim/ccfTemplate.json'

az deployment group what-if --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME"
az deployment group create --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --mode Incremental --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME"
```

### ARM-template connector

Provide the exact name of the existing Cato table and use the ARM template:

```bash
TABLE_NAME='<existing-table-name>'
TEMPLATE_URI='https://raw.githubusercontent.com/catonetworks/cato-sentinel/a26a985e510b0472d974cceb3e747df2e21ac75d/asim/armTemplate.json'

az deployment group what-if --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME" tableName="$TABLE_NAME"
az deployment group create --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --mode Incremental --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME" tableName="$TABLE_NAME"
```

## Query and integrate

Functions are query-time saved searches over the chosen source table. For example, for a custom table named `MyCatoEvents_CL`:

```kusto
vimNetworkSessionCato_MyCatoEvents_CL(starttime=ago(1d))
```

For the CCF `CatoNetworksEvents_CL` table:

```kusto
vimNetworkSessionCato_CatoNetworksEvents_CL(starttime=ago(1d))
```

You may optionally integrate a source-function invocation into the applicable ASIM unifier, following [Microsoft's ASIM parser management guidance](https://learn.microsoft.com/en-us/azure/sentinel/normalization-manage-parsers). Preserve existing sources and customized unifiers; do not overwrite built-in or other providers' definitions. The installer does not modify unifiers. Parser updates are manual and independent of ingestion-schema updates.

A query returning no rows does not by itself indicate an installation failure. The selected time range may have no matching events, or the parser may lack the source fields or values required for output. Dashboards likewise need matching events, populated fields, and a suitable time range.

## Customizations and rollback

Review dependent queries, workbooks, and customizations before upgrading: catalog versions can change identifiers and mapped field roles. Back up customized managed functions before deployment, and keep local forks outside Cato's aliases to avoid collisions. To roll back, redeploy the prior approved template version in Incremental mode. Incremental deployment does not remove functions absent from that template; review dependencies and unifier registrations before manually removing functions.

## Customer references

- [Cato Event Schema](https://knowledge.catonetworks.com/docs/cato-event-schema)
- [EventFieldName in the Cato API documentation](https://api.catonetworks.com/documentation/#definition-EventFieldName)
- [Integrating Cato Events with Microsoft Sentinel](https://knowledge.catonetworks.com/docs/integrating-cato-events-with-microsoft-sentinel)
