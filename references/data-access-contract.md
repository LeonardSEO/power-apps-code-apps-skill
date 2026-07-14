# Generated-service & data-access contract

"Use the generated services" is not enough — this is the concrete contract for calling
Dataverse, flows, file columns and connectors from app code. All of it lives behind
`src/generated/**`; keep these calls inside a repository/service layer, never in components,
and never hand-edit generated files.

## Generated Dataverse service surface

Every generated `Ths_xService` / `AccountsService` exposes the same shape:

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
  await Ths_beheerdagsService.create({
    ths_titel: 'X',
    'ths_Klant@odata.bind': `/ths_pdpklantens(${guid})`,   // nav prop + plural set
  } as Omit<Ths_beheerdagsBase, 'ths_beheerdagid'>);
  ```

- **Read a lookup**: the GUID is on `_<field>_value`; the display text is on `<field>name`.
  A lookup value is a GUID — rendering it directly is the classic "column shows a GUID" bug.
  Resolve it (join to the target record, or select the formatted `…name`), don't display
  `_value`.
- **Option sets / choices** are numeric. Use the constants from the generated model; never
  invent numeric values. Read the label from the matching `…name` field.

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
a client-side check is not a guarantee. Add the server-side Custom API and call it via the
generated service (see [dataverse-provisioning.md](dataverse-provisioning.md) —
`add-dataverse-api --api-name`). The app then calls one generated action instead of a
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
