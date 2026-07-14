# Dataverse Provisioning and Runtime Design

## Contents
- Decision ladder
- What CLI can do directly
- What still needs solution import or Web API
- Step-by-step build order
- Code snippets for generated services
- Web API metadata fallback
- Backend repository pattern
- RoomMinutes table example

## Decision ladder
Before adding Dataverse to a Code App, lock these decisions first:
1. Is Dataverse the real runtime store, or only a target architecture?
2. Which data belongs in Dataverse versus Blob Storage or another file store?
3. Will the frontend use generated services directly, or will the backend own Dataverse access?
4. Is schema deployment done by solution import, or by Web API metadata automation?

Do not blur these decisions together. In RoomMinutes, the environment had Dataverse and the backend had a Dataverse repository, but runtime persistence still used blob-backed metadata because the feature flag and repository choice resolved that way.

## What CLI can do directly

### 1. Create an environment with Dataverse
The PAC CLI supports creating a Dataverse instance.

Simple example:

```bash
pac admin create \
  --name "Contoso Test" \
  --type Sandbox \
  --domain ContosoTest
```

Advanced example:

```bash
pac admin create \
  --name "Contoso Marketing" \
  --type Production \
  --domain ContosoMarketing \
  --region europe \
  --currency EUR
```

### 2. Create a service principal for backend access
Use this when Azure Functions or another backend needs app-user access to Dataverse.

```bash
pac admin create-service-principal --environment <environment-id> --role "System Administrator"
```

Reduce that role later. For production, scope the app user to the smallest set of tables and privileges that actually needs backend access.

### 3. Attach existing Dataverse tables to a Code App
This is the supported generated-service flow for tables:

```bash
pac auth create
pac env select --environment <environment-id>
pac code add-data-source -a dataverse -t rm_meeting
pac code add-data-source -a dataverse -t rm_chatmessage
pac code add-data-source -a dataverse -t rm_useraiprofile
```

This generates model and service files under `src/generated/...`.

### 4. Add Dataverse actions or functions
Classic PAC data-source flow still skips Dataverse operation schema files. The latest npm CLI has a preview route for Dataverse operations:

```bash
power-apps find-dataverse-api --search "WhoAmI"
power-apps add-dataverse-api --api-name <operation-name>   # flag is --api-name (NOT --name)
```

Treat this as newer platform surface area and verify the generated files before you rely on it. This is the sanctioned route for calling a Custom API / action (e.g. an atomic server-side operation) from the app; the generated service is invoked like any other generated service.

## What not to claim
- Do not claim a single PAC CLI flow can always create every custom Dataverse table and column from zero.
- `pac code add-data-source` connects existing tables to the app. It is not a general schema-authoring tool.
- Existing connections are still the default expectation for connector setup in the initial Code Apps data flow.

## Preferred build order
1. Create or select the Power Platform environment.
2. Confirm Dataverse is enabled and note:
   - environment id
   - Dataverse org URL
   - Web API base URL
3. Provision schema by solution import when possible.
4. Add the tables to the Code App with `pac code add-data-source`.
5. Implement CRUD using generated services.
6. If a backend also talks to Dataverse, provision its app user and test token-based access separately.
7. Verify runtime actually uses Dataverse instead of an in-memory or blob fallback.

## Preferred schema path: solution import
Use a solution import when the schema is part of a repeatable application release:

```bash
pac solution import --path ./roomminutes-dataverse.zip
```

Why this is the safer default:
- tables, columns, forms, views, and related metadata move together,
- ALM stays coherent across Dev, Test, and Prod,
- Code App data-source generation then targets a known schema,
- the schema can be versioned outside ad hoc portal clicks.

## Generated-service usage in the Code App
Use the generated services instead of hand-rolled fetches when the Code App itself reads or writes Dataverse data.

```ts
import { RmMeetingService } from "./generated/services/RmMeetingService";
import type { RmMeeting } from "./generated/models/RmMeetingModel";

const created = await RmMeetingService.create({
  rm_title: "Weekly standup",
  rm_status: "recording",
  rm_durationms: 0,
} as Omit<RmMeeting, "rm_meetingid">);

const meetings = await RmMeetingService.getAll({
  select: ["rm_title", "rm_status", "modifiedon"],
  orderBy: ["modifiedon desc"],
  top: 20,
});
```

Rules:
- exclude system-managed fields on create,
- update only the changed columns,
- always use `select`,
- avoid unbounded `getAll()` on real data sets.

## Web API metadata fallback
Use Web API metadata only when you truly need code-driven schema creation and you can tolerate the extra complexity.

Create a custom user-owned table inside a solution:

```bash
curl -X POST "$DATAVERSE_WEB_API/EntityDefinitions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "MSCRM.SolutionUniqueName: roomminutes" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "OData-MaxVersion: 4.0" \
  -H "OData-Version: 4.0" \
  --data '{
    "@odata.type": "Microsoft.Dynamics.CRM.EntityMetadata",
    "Attributes": [
      {
        "@odata.type": "Microsoft.Dynamics.CRM.StringAttributeMetadata",
        "AttributeType": "String",
        "AttributeTypeName": { "Value": "StringType" },
        "SchemaName": "rm_Name",
        "IsPrimaryName": true,
        "RequiredLevel": {
          "Value": "None",
          "CanBeChanged": true,
          "ManagedPropertyLogicalName": "canmodifyrequirementlevelsettings"
        },
        "FormatName": { "Value": "Text" },
        "MaxLength": 100,
        "DisplayName": {
          "@odata.type": "Microsoft.Dynamics.CRM.Label",
          "LocalizedLabels": [
            {
              "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel",
              "Label": "Name",
              "LanguageCode": 1033
            }
          ]
        }
      }
    ],
    "DisplayName": {
      "@odata.type": "Microsoft.Dynamics.CRM.Label",
      "LocalizedLabels": [
        {
          "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel",
          "Label": "RoomMinutes Meeting",
          "LanguageCode": 1033
        }
      ]
    },
    "DisplayCollectionName": {
      "@odata.type": "Microsoft.Dynamics.CRM.Label",
      "LocalizedLabels": [
        {
          "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel",
          "Label": "RoomMinutes Meetings",
          "LanguageCode": 1033
        }
      ]
    },
    "HasActivities": false,
    "HasNotes": false,
    "IsActivity": false,
    "OwnershipType": "UserOwned",
    "SchemaName": "rm_Meeting"
  }'
```

Important:
- use your real solution publisher prefix, not a throwaway prefix,
- metadata updates use `PUT`, not `PATCH`,
- after metadata changes, publish customizations before assuming the app can use the new schema,
- treat this as admin or deployment automation, not as Code App runtime logic.

## Backend repository pattern
If the backend owns Dataverse access, use token-based Web API calls from the server. RoomMinutes already follows this pattern:

```ts
const token = await credential.getToken(`${new URL(baseUrl).origin}/.default`);
const response = await fetch(`${baseUrl.replace(/\/+$/, "")}/api/data/v9.2/rm_meetings`, {
  method: "GET",
  headers: {
    Authorization: `Bearer ${token?.token ?? ""}`,
    Accept: "application/json",
  },
});
```

This is the right place for:
- server-side policy,
- owner binding,
- systemuser lookup,
- durable writes that the client must not bypass.

## RoomMinutes table example
The intended RoomMinutes table set was:
- `rm_meeting`
- `rm_chatmessage`
- `rm_useraiprofile`

Representative meeting fields:
- `rm_title`
- `rm_status`
- `rm_durationms`
- `rm_summarymarkdown`
- `rm_transcripttext`
- `rm_chunkcount`
- `rm_audiomanifestjson`
- `rm_transcriptblobpath`
- `rm_owneraadobjectid`

Use this as a pattern for AI-generated schema design:
- one main meeting table,
- one child or related message table,
- one per-user AI profile table,
- explicit owner-tracking fields when backend filtering or app-user binding needs them.

## Reality check before saying “Dataverse is live”
Confirm all of these:
- the environment has Dataverse,
- the custom tables exist,
- the app or backend can authenticate to Dataverse,
- the runtime repository points to Dataverse,
- the relevant feature flags do not route writes somewhere else,
- new records actually appear in Dataverse after a smoke test.
