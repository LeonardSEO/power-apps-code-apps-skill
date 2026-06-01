# Troubleshooting and Lessons Learned

## Contents
- Preflight checklist
- Release split
- Symptom to cause to fix
- Cache and debug hygiene
- Dataverse reality checks

## Preflight checklist
Collect these facts before building or debugging:
- Node.js 22+ (`node --version` — required, v22 minimum)
- environment id (GUID from make.powerapps.com URL)
- Code App id if it already exists
- Dataverse enabled: yes or no
- Dataverse org URL
- Dataverse Web API URL
- frontend host model: Power Apps only, or Power Apps plus direct backend fetch
- backend host
- blob host
- auth model
- direct `fetch()` or connector
- runtime storage truth: Dataverse, Blob, SQL, in-memory, or mixed

If any of these are unknown, say so explicitly before the implementation starts.

## Build failure recovery

| Error | Fix |
|---|---|
| TS6133 (unused import) | Remove the unused import and retry once |
| TS2322 / other type error | Report the file and line number; fix before deploying |
| Module not found | Run `npm install` in the project root and retry once |
| Node.js version error | Upgrade to v22+ or switch with `nvm use 22` |
| Auth error on push | Run `npx power-apps logout`, then retry — CLI will re-prompt browser login |
| `environment config does not match` | Update `environmentId` in `power.config.json` to the target environment |
| `npx power-apps add-data-source` fails | Report exact error; check connection ID with `npx power-apps list-connections` |

Never deploy if `npm run build` has not succeeded in the current session.

## Release split
Never collapse these into one imagined deploy step:

### Frontend
```bash
npm run build
npx power-apps push
```

### Backend
```bash
func azure functionapp publish <function-app-name> --build remote --typescript
```

### Platform settings
- Power Apps environment CSP
- Function App CORS
- Blob Storage CORS
- connections and connection references

The frontend can be successfully pushed while the backend is still dead. The backend can be healthy while CSP still blocks the browser. Treat them as separate layers.

## Symptom to cause to fix

### White page or 404 on `/assets/...`
Cause:
- Vite built absolute asset paths that break behind the Power Apps storage proxy.

Fix:

```ts
export default defineConfig({
  base: "./",
  plugins: [react()],
});
```

Lesson:
- A Code App is not a normal root-hosted Vite site. Build it for proxied static hosting.

### `connect-src 'none'`
Cause:
- Code Apps default CSP starts with `connect-src 'none'`.

Fix:
- configure Code Apps CSP at the environment level,
- allow the exact backend and storage hosts in `connect-src`.

Lesson:
- Direct browser calls are a platform question first, not just a frontend code question.

### Function App preflight blocked
Cause:
- the real Power Apps content origin is not in Function App CORS.

Fix:
- add the exact content origin that the browser uses,
- do not assume `apps.powerapps.com` is the only relevant origin.

Lesson:
- inspect the failing request origin in DevTools instead of guessing.

### Blob upload fails after chunk URL works
Cause:
- the SAS or upload-url call succeeded,
- but the browser `PUT` to Blob Storage failed on Blob CORS.

Fix:
- configure Blob Storage CORS separately from Function CORS,
- include the right origin and methods such as `GET`, `PUT`, `HEAD`, `OPTIONS`.

Lesson:
- backend CORS and blob CORS are independent.

### `Meeting not found` after create succeeded
Cause:
- meeting state was kept in process memory,
- the next function invocation did not see that state.

Fix:
- move to durable storage such as Blob-backed metadata or Dataverse.

Lesson:
- process memory is not a persistence strategy in Azure Functions.

### `Could not find a registered durable client input binding`
Cause:
- `df.getClient(context)` was called without registering a durable client input binding.

Fix:

```ts
app.http("complete-meeting", {
  methods: ["POST"],
  authLevel: "anonymous",
  route: "meetings/{meetingId}/complete",
  extraInputs: [df.input.durableClient()],
  handler: completeMeetingHandler,
});
```

Lesson:
- Durable Functions on Node v4 require explicit binding registration on the HTTP function.

### Publish succeeded but routes still return 404
Cause:
- deployment built the package incorrectly for Azure runtime,
- or dependencies were not resolved the way the live host needs.

Fix:
- publish with remote build,
- then test a real route instead of trusting a successful publish banner.

Lesson:
- “publish successful” is not equivalent to “runtime healthy.”

### Console full of warnings
Cause:
- host warnings and app warnings are mixed together.

Fix:
- isolate the first failing network request,
- treat unrelated host noise as secondary until proven otherwise.

Lesson:
- noisy consoles are common in hosted platforms. Follow the broken request, not the loudest warning.

## Cache and debug hygiene
- Use a private window when Power Apps caching gets confusing.
- Hard-refresh after pushes.
- Verify the exact content origin and asset URLs in the Network tab.
- When debugging CSP or CORS, keep one browser tab focused on the failed request details.
- Do not assume a frontend code change actually deployed until the new asset hashes or app version are visible.

## Dataverse reality check
Separate these five ideas every time:
1. Dataverse exists in the environment.
2. Custom tables exist.
3. The Code App has generated data sources for those tables.
4. The backend can authenticate to Dataverse.
5. Runtime writes actually go to Dataverse.

RoomMinutes showed why this matters. The backend used this repository choice:

```ts
const repository = env.useInMemory
  ? storage
    ? new BlobBackedRoomMinutesRepository(storage)
    : new InMemoryRoomMinutesRepository()
  : new DataverseRoomMinutesRepository(required(env.dataverseUrl, "RM_DATAVERSE_URL"));
```

Lesson:
- the presence of a Dataverse repository class proves nothing by itself,
- feature flags and runtime wiring decide the actual persistence layer.
