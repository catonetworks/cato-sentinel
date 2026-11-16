# Maintaining Cato connectors for Microsoft Sentinel

This repository provides separate maintenance templates for the Cato Networks Events connector and Cato ASIM parsers. Choose the operation that matches your connector and goal:

**CCF** is Microsoft's Codeless Connector Framework; here it refers to the **Cato Networks Events (Push)** connector installed through the Microsoft Sentinel Content Hub. **ARM-template connector** refers to the Cato connector deployed with a customer-selected table. Both parser installers in this repository are Azure Resource Manager (ARM) template files; choose one based on the connector you use, not the template file format.

| Operation | Template | Applies to | Effect / required parameter |
|---|---|---|---|
| Update ingestion schema | [`updateTemplate.json`](updateTemplate.json) | CCF connector only; fixed table `CatoNetworksEvents_CL` | Updates the custom table schema and Data Collection Rule (DCR); enter `workspace`. |
| Install/update ASIM parsers | [`asim/ccfTemplate.json`](asim/ccfTemplate.json) | CCF connector; fixed table `CatoNetworksEvents_CL` | Deploys parser saved searches only; enter `workspaceName`. |
| Install/update ASIM parsers | [`asim/armTemplate.json`](asim/armTemplate.json) | ARM-template connector; an existing customer-selected table | Deploys parser saved searches only; enter `workspaceName` and `tableName`. |

The schema updater is **not** an ARM connector schema-update tool. The parser templates work with both connector flavors and do not change the table schema or DCR. Schema and parser updates are independent; neither updates the other. Use Azure Resource Manager **Incremental** deployment mode for these templates, never Complete mode.

## Update the CCF connector schema

Use `updateTemplate.json` for the CCF connector's `CatoNetworksEvents_CL` table when you need the current event schema. It updates the existing table and DCR transformation. It does not install or update parser functions, recreate the Entra application, rotate the client secret, or rewrite already-ingested data.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel-ccf%2Fmain%2FupdateTemplate.json)

1. Select the subscription and resource group containing the Sentinel workspace.
2. Enter the workspace name in the `workspace` parameter.
3. Review and create the deployment.

The Azure region setting is a standard deployment setting; the DCR retains its existing region. Apply the update separately in each workspace where you use the connector. Allow up to 30 minutes for the ingestion endpoint to reflect the updated schema. You can check recent events with:

```kusto
CatoNetworksEvents_CL
| where TimeGenerated > ago(1h)
| take 10
```

New fields may be empty during propagation or when the source event does not populate them. If expected fields remain empty after an hour, contact Cato support.

## Install or update ASIM parsers

The parser templates create or update workspace saved searches only. Use the CCF template for the fixed `CatoNetworksEvents_CL` table, or the ARM template for an existing custom table. See the [ASIM parser guide](asim/README.md) for portal and CLI steps, requirements, query examples, compatibility notes, and current preview limitations.

## Event field reference and support

The `CatoNetworksEvents_CL` fields mirror the Cato events feed, with two names adapted for Log Analytics: `time` is `TimeGenerated`, and `title` is `activity_title`.

- [Cato Event Schema](https://knowledge.catonetworks.com/docs/cato-event-schema)
- [EventFieldName in the Cato API documentation](https://api.catonetworks.com/documentation/#definition-EventFieldName)
- [Integrating Cato Events with Microsoft Sentinel](https://knowledge.catonetworks.com/docs/integrating-cato-events-with-microsoft-sentinel)
- [Cato Knowledge Base](https://knowledge.catonetworks.com/)
- [Cato Networks support](https://support.catonetworks.com/)
