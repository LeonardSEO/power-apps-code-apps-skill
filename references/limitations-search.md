# Limitations and Search Rules

## Important limits and gotchas
- **Node.js 22+ is required.** `npx power-apps add-data-source` rejects Node 20 and earlier.
- **Direct HTTP calls do not work.** The Power Apps sandbox blocks arbitrary outbound fetch/axios calls. Use connector-proxied calls only.
- Published code is hosted on a publicly accessible endpoint. Do not store sensitive user or organizational data in the app bundle.
- Code Apps are not supported in the Power Apps mobile app or Power Apps for Windows.
- Power BI integration through `PowerBIIntegration` is not supported, though embedding in Power BI reports through the Power Apps visual is possible.
- SharePoint forms integration is not supported.
- Power Platform Git integration is not supported for Code Apps.
- Storage SAS IP restriction is not yet supported.
- ALM currently lacks solution packager support and source code integration.
- Local development can be blocked by Chrome or Edge Local Network Access restrictions.

## Use the bundled PDF correctly
The repository PDF is a snapshot of the Code Apps documentation. Use it for orientation, but treat Microsoft Learn as the current source of truth because Code Apps are moving quickly.

## Search order
When web search is available, search official Microsoft Learn pages first.

Start here:
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/overview
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/architecture
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/content-security-policy
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/npm-quickstart
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/connect-to-data
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/connect-to-dataverse
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/add-dataverse-action-function
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/sharepoint-operations
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/connect-to-copilot-studio
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/create-basic-asset-management-api-azure-functions
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/alm
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/troubleshoot-add-datasource
- https://learn.microsoft.com/en-us/power-apps/developer/code-apps/system-limits-configuration
- https://learn.microsoft.com/en-us/power-platform/developer/cli/introduction
- https://learn.microsoft.com/en-us/connectors/custom-connectors/create-custom-connector-aad-protected-azure-functions
- https://learn.microsoft.com/en-us/connectors/custom-connectors/use-custom-connector-powerapps
- https://learn.microsoft.com/en-us/power-apps/maker/data-platform/create-connection-reference
- https://learn.microsoft.com/en-us/power-apps/developer/data-platform/custom-api
- https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/create-update-entity-definitions-using-web-api
- https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview

## Search behavior
- Use Microsoft Learn first for facts about commands, support, and limitations.
- Use the newest Learn article available when two pages overlap.
- If you must fall back to GitHub, blogs, or forum posts, label them as non-official.
- If a feature is preview, say so explicitly.

## What not to claim without checking
- automatic deployment of Azure Functions from `power-apps push`,
- full SharePoint document-library processing support,
- custom auth requirements inside the app,
- mobile app support,
- connector schema refresh commands that do not exist,
- universal one-command Dataverse schema creation via PAC CLI alone,
- Dataverse generated-service support for unsupported classic PAC features such as FetchXML,
- or blanket statements about Dataverse actions and functions without checking whether the user is on the latest npm CLI preview flow.
