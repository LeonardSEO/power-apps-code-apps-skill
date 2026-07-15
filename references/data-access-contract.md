# Generated-service & data-access contract

"Use the generated services" is not enough — this is the concrete contract for calling
Dataverse, flows, file columns and connectors from app code. All of it lives behind
`src/generated/**`; keep these calls inside a repository/service layer, never in components,
and never hand-edit generated files.

## Generated Dataverse service surface

Every generated `<Table>Service` / `AccountsService` exposes the same shape:

```ts
create(record)                          // → IOperationResult<T>
get(id, { select })                     // → IOperationResult<T>
getAll({ select, filter, orderBy, top }) // → IOperationResult<T[]>
update(id, changedFields)               // → IOperationResult<T>   (partial update)
delete(id)                              // → Promise<void>   ← NOT IOperationResult, throws on failure
getMetadata()
```

**Watch the `delete` asymmetry:** `create/update/get/getAll` return `IOperationResult`
(`.success` / `.data` / `.error`); `delete` returns `Promise<void>` and throws. Checking
`result.success` on a `delete` is a compile error.

```ts
const res = await AccountsService.create({ name: 'X' } as Omit<Accounts, 'accountid'>);
if (!res.success || !res.data) throw new Error(formatError(res.error)); // never surface raw JSON
```

## Lookups: `@odata.bind`, `_value`, and `…name`

- **Set a lookup** on create/update with the PascalCase navigation property + `@odata.bind`
  pointing at the **plural entity-set** name:

  ```ts
  await <Table>Service.create({
    <prefix>_name: 'X',
    '<prefix>_Parent@odata.bind': `/<prefix>_parents(${guid})`,   // nav prop + plural set
  } as Omit<<Table>Base, '<prefix>_<table>id'>);
  ```

- **Read a lookup**: the GUID is on `_<field>_value`; the generated model also declares a
  `<field>name` pseudo-field for the display text — but **do not put `<field>name` in
  `select`**. It is not a real column; selecting it throws `Could not find a property named
  '<field>name' on type ...`. It is *also not auto-populated* just because `_<field>_value`
  is in `select` — that assumption silently returns `undefined`/a fallback string instead of
  throwing, so it's easy to ship "every name says Unknown" and not notice until someone
  points at the screen. The actual fix: batch-resolve the GUIDs yourself with a follow-up
  query (one call per page of rows, `filter: ids.map(id => \`<idField> eq ${id}\`).join(' or
  ')`), then merge names back onto the mapped objects. Do this once as a shared helper, not
  per call site. Never render `_value` directly — that's the classic "column shows a GUID" bug.
- **`ownerid` and other polymorphic lookups** (systemuser-or-team, etc.) reject a raw scalar
  value on create/update — `ownerid: someGuid` throws `A 'PrimitiveValue' node with non-null
  value was found ... 'StartObject' node ... was expected`. Polymorphic lookups still go
  through `@odata.bind` like any other lookup:
  ```ts
  'ownerid@odata.bind': `/systemusers(${userId})`,   // correct
  ownerid: userId,                                    // wrong — throws at runtime
  ```
  The generated `Base` model still types `ownerid`/`owneridtype` as required plain fields, so
  once you replace them with the `@odata.bind` key the payload needs
  `as unknown as Omit<YourBase, 'yourIdField'>` instead of a direct `as` cast (TypeScript
  refuses a direct cast when too few of the "required" keys are still present).
- **Option sets / choices** are numeric. Use the constants from the generated model; never
  invent numeric values. Read the label from the matching `…name` field (same caveat as
  lookups above — verify it's actually populated for your query shape rather than assuming).
- **`null` vs `undefined`**: Dataverse returns JSON `null` for empty optional columns; the
  generated types say `field?: Type`, which reads as "may be `undefined`". Code written as
  `value === undefined ? fallback : value.toFixed(2)` lets `null` through and crashes on
  `.toFixed()`. This is invisible against mock/example data (which naturally omits the key
  rather than nulling it) and only shows up against real rows. Fix once, at the row →
  domain-object mapping boundary: `amount: row.<prefix>_amount ?? undefined` for every
  optional field — not at each call site.
- **Excluding service/application accounts from a "list people" query**: a plain
  `systemusers` query also returns application users, connector service accounts, and flow
  accounts (`#PowerAutomate-...`, `#PowerPages...`, etc). Add `accessmode eq 0` (Read-Write —
  genuine interactive users) alongside `isdisabled eq false` to filter them out.

## Calling a cloud flow

A flow added via `add-flow` is invoked through the connector client, not HTTP. The input
shape comes from `.power/schemas/logicflows/*.Schema.json` (`ManualTriggerInput`):

```ts
import { getClient } from '@microsoft/power-apps/data';
import { dataSourcesInfo } from '../../.power/schemas/appschemas/dataSourcesInfo';

const client = getClient(dataSourcesInfo);
const result = await client.executeAsync({
  connectorOperation: {
    tableName: 'pa_sluitbeheerdag',       // the flow's data-source name
    operationName: 'Run',
    parameters: { input: { text, number, number_1, number_2 }, 'api-version': '2015-02-01-preview' },
  },
});
```

The generated `PA_<Flow>Service.Run(input)` wraps exactly this. A `ResponseActionOutput` may
carry a business flag (e.g. `{ succes: boolean }`) — check it, not just transport success.

## File / image columns

Dataverse file/image columns are read and written as bytes through the client, not fetched:

```ts
const client = getClient(dataSourcesInfo);
const { data /* Uint8Array */, fileName } = await client.downloadFileFromRecord(table, id, column);
await client.uploadFileToRecord(table, id, column, name, arrayBuffer);
```

Use this for PDFs/images instead of a direct URL. If a feature stores its output in
SharePoint via a flow (not a Dataverse file column) and the account lacks that SharePoint
connection, the feature is blocked until access is granted OR the flow also writes a
Dataverse file column.

## Custom API / action (server-side authority)

For rules that must be atomic/authoritative (e.g. "exactly one active record per parent"),
or for anything that needs to Associate/Disassociate a many-to-many relationship (the
generated client cannot do this at all — `create`/`delete` on tables like `systemuserroles`
or `teammembership` throws `The 'Create' method does not support entities of type '<table>'`),
a client-side check is not a guarantee. If the Custom API doesn't exist yet, provision it —
plugin, assembly, and all — with the fully CLI-driven procedure in
[dataverse-provisioning.md](dataverse-provisioning.md#provisioning-a-custom-api--plugin--100-cli-no-maker-portal).
Once it exists, wire it into the app with `add-dataverse-api --api-name` and call the
generated service like any other — the app then calls one generated action instead of a
read-then-write race.

## Office 365 Users (avatars, profiles) — a distinct connector

"Office 365" is ambiguous. For user profiles/photos you need the **Office 365 Users**
connector (NOT Office 365 Outlook). The generated method is
`Office365UsersService.UserPhoto_V2(idOrEmail)`; convert its result to a browser-usable
(blob/data) URL before binding it to `<img>`. Other useful ops: profile, manager, direct
reports, user search. Inspect the generated file with search rather than loading it whole:

```bash
rg -n "public static async" src/generated/services/Office365UsersService.ts
```
