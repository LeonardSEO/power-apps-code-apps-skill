# Data and Integrations

## Mental model
Code Apps are connector-first. The usual path is:
- create or reuse a connection in Power Apps,
- add the data source to the code app,
- import the generated model and service,
- call the generated TypeScript methods.

Generated files land under:
- `src/generated/models`
- `src/generated/services`

Do not edit those files by hand.

## General connector flow
Typical PAC pattern:

```bash
pac auth create
pac env select --environment <environment-id>
pac code add-data-source -a <apiId> -c <connectionId> ...
```

If the schema changes on the connection, there is currently no typed-model refresh command. Delete the data source and add it again.

## Dataverse
Use Dataverse when you want platform-native storage, security, solution support, and clean ALM.

For environment creation, schema deployment, Web API metadata, generated services, and backend access patterns, read [dataverse-provisioning.md](dataverse-provisioning.md).

Add a table:

```bash
pac code add-data-source -a dataverse -t <table-logical-name>
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

## Copilot Studio agents
You can connect a published Copilot Studio agent as a data source.

Find the connection:

```bash
pac connection list
```

Add the connector:

```bash
pac code add-data-source -a "shared_microsoftcopilotstudio" -c <connectionId>
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
pac code add-data-source --apiid shared_sharepointonline --connectionId <connection-id> --dataset "@envvar:crd1b_SharepointSiteVar" --table "@envvar:crd1b_sharepointList"
```

This keeps the code app portable across environments.

## Performance rules
- Always use `select`, filters, sorting, paging, and `top` where available.
- Avoid `getAll()` on large data sets unless the data set is genuinely small.
- Avoid loops that trigger N connector calls.
- Keep the client thin. Put authoritative or expensive rules elsewhere.
- If the backend, not the client, is the true owner of persistence, say that explicitly and avoid duplicating write paths in both places.
