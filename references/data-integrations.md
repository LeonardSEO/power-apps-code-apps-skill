# Data and Integrations

## Connector-first rule and the direct-HTTP exception

Use generated connector services for Dataverse, Microsoft Graph, Microsoft 365, Azure management APIs, and any service with a supported Power Platform connector.

| MUST NOT | MUST |
|---|---|
| `fetch("https://graph.microsoft.com/...")` | Use the Outlook, SharePoint, or Dataverse guidance below |
| `axios.get("https://dev.azure.com/...")` | Use the Azure DevOps connector guidance below |
| Any raw HTTP call to an M365/Azure service | Use a Power Platform connector |

If no connector supports the required functionality, prefer a custom connector or a protected backend. A direct browser call to a custom backend is an exception, not a fallback shortcut: complete every CSP, CORS, auth, data-sensitivity, governance, and deployed-host check in [backend-security.md](backend-security.md) before implementing it.

## Mental model
Code Apps are connector-first. The usual path is:
- find or create a connection in Power Apps,
- get the connection ID with the resolved CLI,
- add the data source to the code app,
- import the generated model and service,
- call the generated TypeScript methods.

Generated files land under:
- `src/generated/models`
- `src/generated/services`

Do not edit those files by hand.

## General connector flow
Commands below use the current grouped `pa` syntax. Resolve the project-local CLI first and translate both the verb and flags when only the flat `power-apps` CLI is available; see [runbook.md](runbook.md#cli-resolution-pa-preferred-power-apps-fallback).

Discover the connector catalog separately from existing authenticated connections:

```bash
pa connector list --search "<capability>" --json
pa connection list --search "<connector>" --json
```

Use the returned connector identifier (often `shared_*`), not a guessed display name. For an authorized new connection, `pa connection create --connector <connector-id> --display-name "<name>"` supports SSO or an interactive browser consent flow, depending on the connector. Continue once the CLI confirms success and returns a connection ID. If unsupported, use `https://make.powerapps.com/environments/<environment-id>/connections`. Connection creation is a cloud change; follow the user's authorization scope.

Find your connection ID first:

```bash
pa connection list
```

Then add the data source (npm CLI, preferred):

```bash
pa app add data-source --connector <apiId> -c <connectionId> [-d <dataset>] [--table <table>]
```

PAC CLI fallback:

```bash
pac auth create
pac env select --environment <environment-id>
pac code add-data-source -a <apiId> -c <connectionId> ...
```

If the schema changes on the connection, refresh the generated types — do NOT delete-and-re-add by default:

```bash
pa app refresh data-source
pa app refresh data-source --name <name>
```

CLI surfaces change. Verify the resolved version and live help before relying on a remembered command. For the flat CLI, refresh with `power-apps refresh-data-source [--data-source-name <name>]` and remove with `power-apps delete-data-source --api-id <apiId> --data-source-name <name>`.

To discover datasets and tables before adding:

```bash
pa connection list-datasets --connector <apiId> -c <connectionId>
pa connection list-tables --connector <apiId> -c <connectionId> -d <dataset>
```

## Choose the integration by goal

| Goal | Connector API ID | Important setup input |
|---|---|---|
| Dataverse rows, files, images | `dataverse` | table logical name |
| SharePoint lists or libraries | `sharepointonline` | site URL plus list/library internal name |
| Teams messages and conversations | `teams` | team, channel, or conversation identifiers |
| Outlook mail and calendar | `office365` | mailbox/calendar identifiers |
| Excel table rows | `excelonlinebusiness` | OneDrive/SharePoint dataset, file, and table |
| OneDrive files and folders | `onedriveforbusiness` | dataset plus path or file identifier |
| Azure DevOps projects/work items | `azuredevops` | organization URL and project |
| SQL Server / Azure SQL tables or stored procedures | `shared_sql` | existing SQL connection, discovered dataset, table or procedure |
| Other connectors (Office 365 Users/Groups, Azure Blob/Queues, custom APIs) | returned catalog ID | non-tabular operation or discovered dataset/table |
| Work IQ / Microsoft 365 Copilot Chat MCP | `shared_a365copilotchatmcp` | connection plus MCP tool contract |
| Copilot Studio agent | `shared_microsoftcopilotstudio` | published agent and exact agent name |

Confirm the actual API ID in `pa connection list`; connector identifiers can differ between tenants or CLI generations.

## Dataverse
Use Dataverse when you want platform-native storage, security, solution support, and clean ALM.

For environment creation, schema deployment, Web API metadata, generated services, and backend access patterns, read [dataverse-provisioning.md](dataverse-provisioning.md).

Add a table:

```bash
pa app add data-source --connector dataverse --table <table-logical-name>
```

Use the generated service:

```ts
import { AccountsService } from "./generated/services/AccountsService";
import type { Accounts } from "./generated/models/AccountsModel";

const created = await AccountsService.create({
  name: "New Account",
} as Omit<Accounts, "accountid">);

const one = await AccountsService.get("<guid>");

const many = await AccountsService.getAll({
  select: ["name", "accountnumber"],
  filter: "address1_country eq 'USA'",
  orderBy: ["name asc"],
  top: 50,
});

await AccountsService.update("<guid>", { name: "Updated Account" });
await AccountsService.delete("<guid>");
```

Current unsupported scenarios include:
- polymorphic lookups,
- FetchXML,
- alternate keys,
- deleting Dataverse data sources via PAC CLI,
- schema metadata CRUD.

Nuance:
- classic `pac code add-data-source` still does not do schema definition CRUD,
- the latest npm CLI has a preview path for some Dataverse actions and functions via `find-dataverse-api` and `add-dataverse-api`.

## SQL Server / Azure SQL

Use an existing authorized SQL connection; database provisioning is a separate task. Discover exact dataset and table/procedure names instead of guessing server/database syntax:

```bash
pa connection list --search sql --json
pa connection list-datasets --connector shared_sql -c <connection-id>
pa connection list-tables --connector shared_sql -c <connection-id> -d <dataset>
pa connection list-procedures -c <connection-id> -d <dataset>
pa app add data-source --connector shared_sql -c <connection-id> -d <dataset> --table <table>
# For a stored procedure, use --procedure instead of --table:
pa app add data-source --connector shared_sql -c <connection-id> -d <dataset> --procedure <procedure>
```

Inspect the generated procedure service/model for parameters and result sets. Do not assume a procedure has the same CRUD methods as a table, or put SQL credentials in the SPA. Older flat CLI uses `--sql-stored-procedure`/`-sp`; current `pa` uses `--procedure`. Read [Microsoft's SQL guide](https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/connect-to-azure-sql) for provisioning or connector restrictions, translating legacy commands against installed help.

## Generic connectors

For capabilities outside this table, use `pa connector list` and the [official connector catalog](https://learn.microsoft.com/en-us/connectors/connector-reference/). A non-tabular connector normally needs connector ID plus connection ID. A tabular source additionally needs a discovered dataset and table. Inspect only the generated methods/models needed by the feature and `.power/schemas/<connector>/`; do not assume every connector supports dataset discovery. Adding a connector does not grant its backend permissions.

## SharePoint
What is supported well:
- SharePoint lists as data sources
- CRUD on list items
- lookup, choice, and person/group value helpers

What is not first-class here:
- document processing APIs
- permission changes
- sync actions

So use SharePoint directly for list data and document metadata. For actual file-content processing or sensitive document workflows, prefer an extra service layer or custom connector.

Before adding the source, discover the dataset and table rather than guessing. SharePoint display names are not always the generated internal names. Inspect the generated service signature after adding or refreshing the source.

- Column internal names survive display-name changes and may contain encodings such as `_x0020_`; copy the generated property, do not derive it by replacing spaces.
- SharePoint choice values are strings, unlike Dataverse numeric option sets. Multi-value and lookup/person payloads depend on the generated operation: inspect its model before assuming a semicolon string, `{ Id, Value }`, or `FieldId` write shape. A SharePoint site-user ID is not an Entra object ID.
- Existing lists: connect directly. New/extended lists: inspect existing schema and propose only missing lists/columns before an authorized provisioning change. List creation is not an effect of `add data-source`. For admin-side Graph provisioning, follow [create list](https://learn.microsoft.com/en-us/graph/api/list-create?view=graph-rest-1.0) and [create column](https://learn.microsoft.com/en-us/graph/api/list-post-columns?view=graph-rest-1.0), with the required delegated/application permissions. Keep provisioning tokens outside the browser app.

## Teams

Add the Teams connection as a data source, then inspect the generated service for operations such as `PostMessageToConversation`. Do not copy a remembered payload shape: team/channel posting and chat/conversation posting use different identifiers and generated request models.

```bash
pa app add data-source --connector teams -c <connection-id>
rg -n "PostMessage|Conversation|Channel" src/generated/services src/generated/models
```

## Outlook

Use `office365` for Outlook mail and calendar operations. Inspect the generated service for the exact operations available to the authenticated connection, such as sending mail, listing messages, or creating calendar events. Keep mailbox addresses and recipient data out of source-controlled fixtures.

```bash
pa app add data-source --connector office365 -c <connection-id>
rg -n "SendEmail|GetEmails|Calendar|Event" src/generated/services src/generated/models
```

## Excel Online (Business)

Excel connector operations address a workbook and an actual Excel table, not an arbitrary worksheet range. Discover the dataset and table first. For generated `AddRowIntoTable`-style methods, pass the row fields in the exact generated request shape; do not invent an `items` wrapper unless the generated model requires one.

```bash
pa connection list-datasets --connector excelonlinebusiness -c <connection-id>
pa connection list-tables --connector excelonlinebusiness -c <connection-id> -d <dataset>
pa app add data-source --connector excelonlinebusiness -c <connection-id> -d <dataset> --table <table>
```

## OneDrive for Business

Use `shared_onedriveforbusiness` for file/folder operations such as listing a folder, reading metadata/content, and creating a file. Paths, file IDs, and binary content are not interchangeable; inspect the generated models for methods such as `ListFolder`, `GetFileMetadata`, `GetFileContent`, and `CreateFile`.

```bash
pa app add data-source --connector onedriveforbusiness -c <connection-id>
rg -n "ListFolder|GetFile|CreateFile" src/generated/services src/generated/models
```

## Azure DevOps

Use the Azure DevOps connector for projects, queries, and work items. After generation, inspect the exact service method and request model before coding. Some generator/connector combinations have emitted a wrapper body that does not match the runtime operation. Never patch generated code from memory:

1. reproduce the mismatch against the current generated output,
2. compare the service, model, `dataSourceInfo`, and connector schema,
3. prefer upgrading or regenerating when the tool has fixed it,
4. if a temporary generated-file patch is unavoidable, make the smallest deterministic change, document that refresh may overwrite it, and rebuild immediately.

```bash
pa app add data-source --connector azuredevops -c <connection-id> -d <organization-url>
rg -n "WorkItem|Query|HttpRequest|body" src/generated/services src/generated/models
```

## Work IQ / Microsoft 365 Copilot Chat MCP

Use Work IQ for internal Microsoft 365 knowledge/search, not general knowledge, public web, or news. When the workload is explicit (mail, Teams, SharePoint, OneDrive), use its specific connector. The broader `shared_a365mcpservers` connector is a different integration with different generated services.

Use `shared_a365copilotchatmcp` and its generated `WorkIQCopilotMCPService.mcp_m365copilot` operation. Treat MCP initialization as stateless-tolerant: this connector works without an `Mcp-Session-Id`, while sending a client-generated ID on `initialize` can cause `-32001 Session not found`. The reusable wrapper should:

- call `initialize`, optionally `tools/list`, then `tools/call`,
- pass the Copilot Chat prompt under the generated `message` argument,
- preserve and reuse a conversation ID when the response provides one,
- accept JSON, nested JSON-string, and SSE-style result payloads,
- surface connector/MCP errors instead of returning empty success-shaped text.

```bash
pa app add data-source --connector shared_a365copilotchatmcp -c <connection-id>
rg -n "initialize|tools/list|CopilotChat|conversation" src/generated/services src/generated/models
```

## Copilot Studio agents
You can connect a published Copilot Studio agent as a data source.

Find the connection:

```bash
pa connection list
```

Add the connector:

```bash
pa app add data-source --connector shared_microsoftcopilotstudio -c <connection-id>
```

Invoke the generated service:

```ts
import { CopilotStudioService } from "./generated/services/CopilotStudioService";

const response = await CopilotStudioService.ExecuteCopilotAsyncV2({
  message: "Summarize the latest product trends",
  notificationUrl: "https://notificationurlplaceholder",
  agentName: "cr3e1_trendAnalyzer",
});

const text = response.data?.lastResponse;
```

Rules:
- publish the agent first,
- use the exact `agentName`,
- prefer `ExecuteCopilotAsyncV2`,
- expect JSON-string payloads or JSON-string responses in some agents.

## Connection references and environment variables
For ALM, bind to connection references instead of personal connections whenever possible.

Environment variables can also be used in data-source configuration:

```bash
pa app add data-source --connector shared_sharepointonline --connection-ref <reference-logical-name> --dataset "@envvar:crd1b_SharepointSiteVar" --table "@envvar:crd1b_sharepointList"
```

Discover references with `pa connection list-references --solution-id <solution-guid>` and variables with `pa app list-environment-variables`. Use the reference logical name returned by discovery. Verify the variable definitions/current values and reference bindings in every target environment; do not assume CLI process variables (`PA_CLI_*`) create solution variables.

## Performance rules
- Always use `select`, filters, sorting, paging, and `top` where available.
- Avoid `getAll()` on large data sets unless the data set is genuinely small.
- Avoid loops that trigger N connector calls.
- Keep the client thin. Put authoritative or expensive rules elsewhere.
- If the backend, not the client, is the true owner of persistence, say that explicitly and avoid duplicating write paths in both places.
