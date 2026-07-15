# Dataverse Provisioning and Runtime Design

## Contents
- Auth: device code vs interactive (what actually works)
- Decision ladder
- What CLI can do directly
- What still needs solution import or Web API
- Step-by-step build order
- Provisioning a Custom API + plugin — 100% CLI, no Maker Portal
- Known gap: Custom APIs have no auto-generated privilege
- Code snippets for generated services
- Web API metadata fallback
- Backend repository pattern
- Example table set shape for a new domain

## Auth: device code vs interactive (what actually works)
`pac auth create --environment <url>` (normal interactive browser MSAL) is the
session every command below relies on. It can go stale mid-session
(`AADSTS50078: MFA has expired`) — just re-run `pac auth create`, no drama.

**Device-code flow can be blocked tenant-wide by Conditional Access**, seen as:
`You don't have access to this resource... device code, app, or location
restricted by your admin`. This is deliberate tenant policy on the *grant
type*, not fixable by trying a different public client ID or scope. Practical
consequence: never mint your own access token in a hand-rolled script
(Node/Python hitting the Dataverse Web API with a device-code-acquired bearer
token) as an "automation" shortcut — if the tenant blocks device code, that
whole approach is a dead end. Route every Dataverse write through `pac` itself
(`pac solution import`, `pac plugin push` for updates to an *existing*
package) so it reuses the already-blessed interactive session instead of
starting a new auth flow. If `pac solution import`/`export` work, you have a
working CLI path even in a tenant that blocks device code entirely.

## Decision ladder
Before adding Dataverse to a Code App, lock these decisions first:
1. Is Dataverse the real runtime store, or only a target architecture?
2. Which data belongs in Dataverse versus Blob Storage or another file store?
3. Will the frontend use generated services directly, or will the backend own Dataverse access?
4. Is schema deployment done by solution import, or by Web API metadata automation?

Do not blur these decisions together. A project can have Dataverse enabled in the environment and a fully implemented Dataverse repository class, while runtime persistence still resolves to blob-backed or in-memory storage because a feature flag or repository-selection wiring resolved that way.

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
pac code add-data-source -a dataverse -t <prefix>_table1
pac code add-data-source -a dataverse -t <prefix>_table2
```

This generates model and service files under `src/generated/...`.

### 4. Add Dataverse actions or functions
Classic PAC data-source flow still skips Dataverse operation schema files. The latest npm CLI has a preview route for Dataverse operations:

```bash
power-apps find-dataverse-api --search "WhoAmI"
power-apps add-dataverse-api --api-name <operation-name>   # flag is --api-name (NOT --name)
```

Treat this as newer platform surface area and verify the generated files before you rely on it. This is the sanctioned route for calling a Custom API / action (e.g. an atomic server-side operation) from the app; the generated service is invoked like any other generated service.

**This only wires an *existing* Custom API into the app.** It does not create the
Custom API or the plugin that backs it. If `find-dataverse-api --search` finds
nothing, you need to provision it first — see [Provisioning a Custom API +
plugin](#provisioning-a-custom-api--plugin--100-cli-no-maker-portal) below.
The obvious-looking `pac plugin push` command does **not** create a new plugin
registration — it only updates an *existing* plugin package (it requires
`--pluginId` of an already-registered package; passing a fresh GUID fails with
`Entity 'PluginPackage' With Id = ... Does Not Exist`). Don't spend time on it
for a first-time registration.

## What not to claim
- Do not claim a single PAC CLI flow can always create every custom Dataverse table and column from zero.
- `pac code add-data-source` connects existing tables to the app. It is not a general schema-authoring tool.
- Existing connections are still the default expectation for connector setup in the initial Code Apps data flow.
- Do not claim `pac plugin push` handles first-time plugin registration — it only updates an existing plugin package.
- Do not claim an unsigned plugin assembly (`pac plugin init --skip-signing`) is always safe. It is documented as fine for NuGet Plugin *Packages*, but a plain solution-file plugin-assembly import can reject it outright with `Public assembly must have public key token` — default to signing (omit `--skip-signing`) unless you've confirmed the target environment accepts unsigned Sandbox plugins.

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
pac solution import --path ./<solution-name>-dataverse.zip
```

Why this is the safer default:
- tables, columns, forms, views, and related metadata move together,
- ALM stays coherent across Dev, Test, and Prod,
- Code App data-source generation then targets a known schema,
- the schema can be versioned outside ad hoc portal clicks.

The same export → unpack → hand-edit → pack → import cycle also provisions
**Custom APIs, their backing plugin, and security-role privileges** —
Maker Portal is not required for any of it once base schema (tables, columns,
security roles) already exists. Full procedure next.

## Provisioning a Custom API + plugin — 100% CLI, no Maker Portal

Use this whenever a rule needs to be atomic/authoritative (see "Custom API /
action" in [data-access-contract.md](data-access-contract.md)) or the app
needs to write a many-to-many relationship. **The generated Code App client
cannot Associate/Disassociate on N:N tables at all** — `systemuserroles`,
`teammembership`, and similar intersect tables reject `create`/`delete`
outright with `The 'Create' method does not support entities of type
'<table>'`. This is a hard Dataverse Web API restriction (confirmed in
Microsoft's own docs), not a client bug — the only fix is a server-side Custom
API backed by a plugin that calls `IOrganizationService.Associate`/
`Disassociate`.

### 1. Scaffold and write the plugin

```bash
pac plugin init --outputDirectory power-platform/plugins/<Name> --author "<you>"
```

Do **not** pass `--skip-signing` by default (see "What not to claim" above).
Delete the generated placeholder class, write a real one extending
`PluginBase`, reading `context.InputParameters["X"]` for each parameter
(names must match the Custom API's parameter `UniqueName`s exactly), and using
`localPluginContext.InitiatingUserService` to act as the calling user. For an
Associate/Disassociate-backed action:

```csharp
service.Associate(entityLogicalName, id,
    new Relationship("<n:n relationship schema name>"),   // e.g. "systemuserroles_association" for user↔role
    new EntityReferenceCollection { new EntityReference(otherEntity, otherId) });
```

Build it:

```bash
cd power-platform/plugins/<Name>
dotnet build -c Release
```

This produces `bin/Release/net462/<Name>.dll` (needed below) and a `.nupkg`
(only relevant if you later get a Plugin *Package* import working — not used
in this route).

### 2. Get the assembly's real identity — don't guess it

Dataverse needs the exact `Name, Version, Culture, PublicKeyToken` .NET
reflection reports. Ask .NET instead of hand-computing it:

```bash
mkdir -p /tmp/asminfo && cd /tmp/asminfo
cat > Program.cs <<'EOF'
using System;
using System.Reflection;
var name = AssemblyName.GetAssemblyName(args[0]);
Console.WriteLine("Name=" + name.Name);
Console.WriteLine("Version=" + name.Version);
Console.WriteLine("CultureName=" + (string.IsNullOrEmpty(name.CultureName) ? "neutral" : name.CultureName));
var pkt = name.GetPublicKeyToken();
Console.WriteLine("PublicKeyToken=" + (pkt == null || pkt.Length == 0 ? "null" : BitConverter.ToString(pkt).Replace("-", "").ToLowerInvariant()));
EOF
cat > asminfo.csproj <<'EOF'
<Project Sdk="Microsoft.NET.Sdk"><PropertyGroup><OutputType>Exe</OutputType><TargetFramework>net9.0</TargetFramework></PropertyGroup></Project>
EOF
dotnet run -- "/absolute/path/to/<Name>.dll"
```

Keep the `PublicKeyToken` — it's needed verbatim in two places below.

### 3. Export, unpack, and inspect the target solution

```bash
pac solution list                                  # find the exact UniqueName
pac solution export --name <SolutionUniqueName> --path ./export.zip --managed false
pac solution unpack --zipfile ./export.zip --folder ./unpacked --packagetype Unmanaged
```

**Inspect what's already there before adding anything** —
`unpacked/PluginAssemblies/`, `unpacked/customapis/`, and
`unpacked/Other/Solution.xml`'s `<RootComponents>`. Solutions accumulate
half-finished attempts; extend, don't silently duplicate or clobber.

Relevant shapes after unpack:

```
unpacked/Other/Solution.xml                                        # manifest + RootComponents + version
unpacked/PluginAssemblies/<Name>-<AssemblyGUID>/<Name>.dll
unpacked/PluginAssemblies/<Name>-<AssemblyGUID>/<Name>.dll.data.xml
unpacked/customapis/<uniquename>/customapi.xml
unpacked/customapis/<uniquename>/customapirequestparameters/<ParamName>/customapirequestparameter.xml
unpacked/Roles/<Role Display Name>.xml
```

### 4. Add the plugin assembly

Pick a fresh GUID for the assembly and one for its plugin type (Dataverse
accepts client-specified GUIDs on solution import). Copy the **signed** DLL
into a new folder, then write `<Name>.dll.data.xml` next to it:

```xml
<?xml version="1.0" encoding="utf-8"?>
<PluginAssembly FullName="<Name>, Version=1.0.0.0, Culture=neutral, PublicKeyToken=<token>" PluginAssemblyId="<assembly-guid-lower>" CustomizationLevel="1" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <IsolationMode>2</IsolationMode>          <!-- 2 = Sandbox -->
  <SourceType>0</SourceType>                <!-- 0 = Database -->
  <IntroducedVersion>1.0</IntroducedVersion>
  <FileName>/PluginAssemblies/<Name>-<ASSEMBLY_GUID_UPPER>/<Name>.dll</FileName>
  <PluginTypes>
    <PluginType AssemblyQualifiedName="<Namespace>.<ClassName>, <Name>, Version=1.0.0.0, Culture=neutral, PublicKeyToken=<token>" PluginTypeId="<plugintype-guid-lower>" Name="<Namespace>.<ClassName>">
      <FriendlyName><ClassName></FriendlyName>
    </PluginType>
  </PluginTypes>
</PluginAssembly>
```

### 5. Complete (or create) the Custom API

`customapis/<uniquename>/customapi.xml` — if a shell already exists, just add
the `<plugintypeid>` block; otherwise write the whole file:

```xml
<customapi uniquename="<prefix>_YourAction">
  <allowedcustomprocessingsteptype>0</allowedcustomprocessingsteptype>  <!-- 0 None, 1 Async Only, 2 Sync and Async -->
  <bindingtype>0</bindingtype>                                          <!-- 0 Global, 1 Entity, 2 EntityCollection -->
  <description default="..."><label description="..." languagecode="1033" /></description>
  <displayname default="..."><label description="..." languagecode="1033" /></displayname>
  <iscustomizable>1</iscustomizable>
  <isfunction>0</isfunction>
  <isprivate>0</isprivate>
  <name>Your Action</name>
  <workflowsdkstepenabled>0</workflowsdkstepenabled>
  <plugintypeid><plugintypeid><plugintype-guid-lower></plugintypeid></plugintypeid>
</customapi>
```

Each parameter gets its own folder (`customapirequestparameters/<Name>/customapirequestparameter.xml`):

```xml
<customapirequestparameter uniquename="UserId">
  <description default="..."><label description="..." languagecode="1033" /></description>
  <displayname default="UserId"><label description="UserId" languagecode="1033" /></displayname>
  <iscustomizable>1</iscustomizable>
  <isoptional>0</isoptional>
  <logicalentityname />
  <name><customapi-uniquename>.UserId</name>
  <type>12</type>
</customapirequestparameter>
```

`type` enum (shared by request parameters and response properties): `0`
Boolean, `1` DateTime, `2` Decimal, `3` Entity, `4` EntityCollection, `5`
EntityReference, `6` Float, `7` Integer, `8` Money, `9` Picklist, `10` String,
`11` StringArray, `12` Guid.

### 6. Register the assembly, bump the version

In `Other/Solution.xml`'s `<RootComponents>`, add one line for the assembly
(type `91` = Plugin Assembly):

```xml
<RootComponent type="91" id="{<assembly-guid-lower>}" schemaName="<Name>, Version=1.0.0.0, Culture=neutral, PublicKeyToken=<token>" behavior="0" />
```

**PluginType and CustomAPI/CustomAPIRequestParameter do not need their own
`<RootComponent>` entries** — only the assembly (type 91) and, if you register
a classic `SdkMessageProcessingStep` on an OOB message (type 92), need
explicit root-component lines. Bump `<Version>` in the same file
(`1.0.0.0` → `1.0.0.1`) — re-importing an unchanged version can warn "already
installed".

### 7. Pack and import

```bash
pac solution pack --folder ./unpacked --zipfile ./out.zip --packagetype Unmanaged
pac solution import --path ./out.zip --publish-changes
```

`pack` echoes every plugin assembly found (`Processing Component:
SolutionPluginAssemblies`) — confirm the new one is listed before importing.
This reuses the same interactive `pac` session as everything else, so it is
unaffected by any Conditional Access block on device code.

### 8. Wire the Custom API into the Code App

```bash
power-apps find-dataverse-api --search <ActionUniqueName>
power-apps add-dataverse-api --api-name <ActionUniqueName>
```

Generates a typed service (`executeAsync` under the hood, `action:
'customapi'`) exactly as described earlier in this file — call it like any
other generated service and check `.success`.

## Known gap: Custom APIs have no auto-generated privilege

Confirmed in Microsoft's own Custom API FAQ: **there is currently no supported
way to create a new privilege just for a Custom API.** `ExecutePrivilegeName`
must reference an *existing* privilege — either reuse an OOB one (rarely
clean, since OOB privileges tend to be broadly held across many roles) or use
the documented workaround: create a throwaway table purely to "borrow" its
auto-generated CRUD privilege (e.g. `prvCreate<prefix>_placeholder`), granting
that privilege only to the role that should be allowed to call the API.

If you don't set `ExecutePrivilegeName`, the Custom API is callable by anyone
with basic Dataverse access — any "only role X can do this" gating is then
UI-only (`checkPermission(...)`), not platform-enforced. Decide the
`ExecutePrivilegeName` story (placeholder table, or an accepted UI-only gate
for a low-stakes internal tool) at design time, not as an afterthought once
the Custom API already works functionally.

## Generated-service usage in the Code App
Use the generated services instead of hand-rolled fetches when the Code App itself reads or writes Dataverse data.

```ts
import { <Table>Service } from "./generated/services/<Table>Service";
import type { <Table> } from "./generated/models/<Table>Model";

const created = await <Table>Service.create({
  <prefix>_title: "Example title",
  <prefix>_status: "active",
} as Omit<<Table>, "<prefix>_<table>id">);

const rows = await <Table>Service.getAll({
  select: ["<prefix>_title", "<prefix>_status", "modifiedon"],
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
  -H "MSCRM.SolutionUniqueName: <solution-unique-name>" \
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
        "SchemaName": "<prefix>_Name",
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
          "Label": "<Prefix> Record",
          "LanguageCode": 1033
        }
      ]
    },
    "DisplayCollectionName": {
      "@odata.type": "Microsoft.Dynamics.CRM.Label",
      "LocalizedLabels": [
        {
          "@odata.type": "Microsoft.Dynamics.CRM.LocalizedLabel",
          "Label": "<Prefix> Records",
          "LanguageCode": 1033
        }
      ]
    },
    "HasActivities": false,
    "HasNotes": false,
    "IsActivity": false,
    "OwnershipType": "UserOwned",
    "SchemaName": "<prefix>_Record"
  }'
```

Important:
- use your real solution publisher prefix, not a throwaway prefix,
- metadata updates use `PUT`, not `PATCH`,
- after metadata changes, publish customizations before assuming the app can use the new schema,
- treat this as admin or deployment automation, not as Code App runtime logic.

## Backend repository pattern
If the backend owns Dataverse access, use token-based Web API calls from the server:

```ts
const token = await credential.getToken(`${new URL(baseUrl).origin}/.default`);
const response = await fetch(`${baseUrl.replace(/\/+$/, "")}/api/data/v9.2/<prefix>_records`, {
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

## Example table set shape for a new domain
A representative shape for a new domain's schema, as a pattern rather than a
specific project's tables:
- one main "parent" table (`<prefix>_<entity>`) holding the core record,
- one child or related table (`<prefix>_<childentity>`) for line items or
  related records where the relationship is one-to-many,
- one per-user table (`<prefix>_userprofile`) if the feature needs per-user
  settings or state distinct from the Dataverse `systemuser` record.

Representative fields to consider on the parent table:
- a primary name/title field,
- a status/state field (picklist),
- explicit owner-tracking fields (e.g. `ownerid`, or `<prefix>_owneraadobjectid`
  if the owner is an external identity rather than a Dataverse user) when
  backend filtering or app-user binding needs them,
- large text or blob-reference fields kept on a child table rather than the
  parent row if they can grow large (long-form text, file/blob path
  references).

## Reality check before saying “Dataverse is live”
Confirm all of these:
- the environment has Dataverse,
- the custom tables exist,
- the app or backend can authenticate to Dataverse,
- the runtime repository points to Dataverse,
- the relevant feature flags do not route writes somewhere else,
- new records actually appear in Dataverse after a smoke test.
