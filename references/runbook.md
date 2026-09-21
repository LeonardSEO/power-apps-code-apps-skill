# Power Apps Code Apps Runbook

## Default stack
- Vite SPA
- TypeScript
- `@microsoft/power-apps`
- `@microsoft/power-apps-cli` for the grouped `pa` CLI; legacy flat `power-apps` only when already installed
- `@microsoft/power-apps-vite` when the project uses its Vite plugin
- PAC CLI for solution/environment/plugin operations and confirmed Code Apps compatibility gaps

## Preflight
Before starting, collect:
- A supported Node.js LTS release installed (`node --version`); Node.js 22 is the current platform baseline and a repository may require a newer minimum
- environment id (GUID from the make.powerapps.com URL: `https://make.powerapps.com/environments/<env-id>/home`)
- Dataverse enabled: yes or no
- backend host
- blob host if direct uploads exist
- auth model
- direct `fetch()` versus connector
- runtime storage truth

## CLI resolution: `pa` preferred, `power-apps` fallback

Resolve from the project root once per session. Never let `npx` install a
missing `pa` or `power-apps` package implicitly.

```bash
if [[ -e node_modules/.bin/pa || -e node_modules/.bin/pa.cmd ]]; then
  PA_KIND=pa
  PA=(npx --no-install pa)
elif [[ -e node_modules/.bin/power-apps || -e node_modules/.bin/power-apps.cmd ]]; then
  PA_KIND=power-apps
  PA=(npx --no-install power-apps)
elif command -v pa >/dev/null 2>&1; then
  PA_KIND=pa
  PA=(pa)
elif command -v power-apps >/dev/null 2>&1; then
  PA_KIND=power-apps
  PA=(power-apps)
else
  PA_KIND=none
  PA=()
fi
```

In native Windows PowerShell, use the equivalent local-shim probe:

```powershell
if (Test-Path node_modules/.bin/pa.cmd) {
  $PaKind = "pa"; $Pa = @("npx", "--no-install", "pa")
} elseif (Test-Path node_modules/.bin/power-apps.cmd) {
  $PaKind = "power-apps"; $Pa = @("npx", "--no-install", "power-apps")
} else {
  $PaKind = "none"; $Pa = @()
}
```

If the repository defines `make code-*` or equivalent package scripts, inspect and reuse them. This standalone skill does not supply those targets. In PowerShell, execute the corresponding resolved form, e.g. `npx --no-install pa app --help` when `$PaKind` is `pa`; translate verbs and flags when it is `power-apps`.

- Prefer the project-local shim. A global official binary is an existing-tool
  fallback, not permission to install globally.
- If `PA_KIND=none`, inspect `package.json` and the lockfile, then choose exactly the matching case:
  - CLI already declared: restore the declared dependencies using the repository's package manager when permitted, then re-probe.
  - CLI absent from dependencies (including SDK 1.3.1-only projects): the next step is a project-local CLI dev dependency, `npm install --save-dev @microsoft/power-apps-cli`, subject to the user's dependency-change approval rules. Reinstalling the SDK is not a step in this case.
  - Ask before any global install.
- Package snapshot checked 2026-09-15: SDK `@microsoft/power-apps` 1.3.1 has no CLI dependency; CLI 1.0.1 requires Node >=22 and exports only `pa`. Older packages may export `power-apps`, or both. Select by actual installed binaries, not the SDK version alone.
- Commands in this reference use canonical grouped `pa` syntax. Execute them as
  `"${PA[@]}" <noun> <verb> ...` when `PA_KIND=pa`. When only `power-apps` is
  available, translate both the verb path and renamed flags before execution.
- Verify unfamiliar or preview flags with `"${PA[@]}" --help` and the relevant
  command help from the installed CLI.

| Operation | Grouped `pa` | Flat `power-apps` |
|---|---|---|
| Initialize | `pa app init` | `power-apps init` |
| Local Play runtime | `pa app run` | `power-apps run` |
| Push | `pa app push` | `power-apps push` |
| List apps | `pa app list` | `power-apps list-codeapps` |
| Add data source | `pa app add data-source` | `power-apps add-data-source` |
| Refresh data source | `pa app refresh data-source` | `power-apps refresh-data-source` |
| Remove data source | `pa app remove data-source` | `power-apps delete-data-source` |
| Find/add Dataverse API | `pa app find-dataverse-api` / `pa app add dataverse-api` | `power-apps find-dataverse-api` / `power-apps add-dataverse-api` |
| List/add/remove flow | `pa app list-flows` / `pa app add flow` / `pa app remove flow` | `power-apps list-flows` / `power-apps add-flow` / `power-apps remove-flow` |
| Connector catalog | `pa connector list` | `power-apps list-connectors` |
| SQL procedures | `pa connection list-procedures` | `power-apps list-sqlStoredProcedures` |
| Connections | `pa connection list` | `power-apps list-connections` |
| Create connection | `pa connection create` | `power-apps create-connection` |
| Datasets/tables | `pa connection list-datasets` / `pa connection list-tables` | `power-apps list-datasets` / `power-apps list-tables` |
| Auth | `pa auth login/status/switch/logout` | `power-apps login/auth-status/auth-switch/logout` |

The grouped discovery paths above are verified for CLI 1.0.1. Older grouped versions used `pa connector list-datasets/list-tables/list-procedures`; use that form only if the installed help exposes it. `pa connection list-procedures` takes connection and dataset, not `--connector`.

Flat mappings apply only to legacy versions that expose those commands. New operations such as sharing and solution discovery are not guaranteed to exist in a flat-only install; check its help instead of inventing a translation.

Renamed selector flags:

| Meaning | Grouped `pa` | Flat `power-apps` |
|---|---|---|
| Connector/API | `--connector` | `--api-id` / `-a` |
| Table/resource | `--table` | `--resource-name` / `-t` |
| Data-source name | `--name` | `--data-source-name` / `-n` |
| SQL procedure to add | `--procedure` | `--sql-stored-procedure` / `-sp` |
| Connection reference | `--connection-ref` | `--connection-ref` / `-cr` |

Connection (`-c`), dataset (`-d`), environment (`-e`) and init display-name
flags are unchanged. The meaning of `-n` is command-specific; prefer the long
form in generated instructions.

## New app bootstrap
Use the official Microsoft template first.

```bash
npx degit github:microsoft/PowerAppsCodeApps/templates/vite my-app
cd my-app
npm install
# If the CLI is absent, arrange its project-local installation as described above.
# Resolve PA before running the following canonical operations.
pa app init --display-name "My Code App" --environment-id <environment-id>
pa app run
```

Notes:
- Scaffold into a new directory; inspect an existing project before adding or replacing template files.
- SDK v1.0.4 introduced the npm CLI route. Current releases separate SDK, CLI, and Vite plugin; preserve the project's compatible versions and check each package independently.
- The `init` command opens a browser window for Microsoft sign-in on first run. Complete login and the command continues. No separate auth setup needed.
- The environment ID is the GUID in the make.powerapps.com URL: `https://make.powerapps.com/environments/<env-id>/home`. If you omit `-e`, the CLI will prompt for it interactively.
- Open the `Local Play` URL in the same browser profile as the Power Platform tenant.
- If local play fails in Chrome or Edge, check Local Network Access restrictions before changing code.

## Existing SPA migration
Use this default sequence:
1. Confirm the app is already a browser SPA, or can be reduced to one.
2. Remove server-only framework assumptions from app code.
3. Keep the app in Vite or move it to the Vite template shell.
4. Add Power Apps SDK init and `power.config.json`.
5. Rewire data access to Power Platform connectors or Dataverse generated services.
6. Push only after the app runs locally in Local Play.

## Maintenance and package hygiene
Use the least risky upgrade flow first.

```bash
npm outdated
npm update
npm audit
```

Rules:
- Prefer targeted upgrades over blanket force upgrades.
- Use `npm audit fix --force` only when the user accepts possible major-version churn and you intend to re-test the app.
- Prefer targeted `overrides` for a known transitive issue instead of repeated `--force` installs.
- Re-run `npm run build` after dependency changes.

## Local Play
Inspect `package.json` and `vite.config.*` first; the current Microsoft template registers `powerApps()` from `@microsoft/power-apps-vite/plugin`.

| Project configuration | Start path |
|---|---|
| Vite plugin registered | `npm run dev` serves app + Power Apps config and prints Local Play. `pa app run` can also launch the dev script and detect the plugin; choose one launcher. |
| No Vite plugin; CLI manages dev script | `pa app run --port 8080 --local-app-url http://localhost:5173` |
| No Vite plugin; dev server already managed separately | `pa app run --config-only --port 8080 --local-app-url http://localhost:5173` if installed help supports `--config-only` |

- Without the Vite plugin, `--port` is the separate config runtime and `--local-app-url` is the frontend URL. With the plugin, config and frontend are served together; do not require a second listener on 8080.
- Use the actual dev-server port from project configuration/output. Do not start a second Vite process or make a `dev` script call `pa app run` recursively.
- Let the CLI print the `Local Play` URL and open THAT. Do not hand-build a URL from an
  old app id — that yields `Launch App failed with Http status code of 0`.
- Open it in the **same browser profile** as the Power Platform tenant. If it fails in
  Chrome/Edge, check **Local Network Access** permissions before touching code.
- `EADDRINUSE :::8080` (or 5173) means the port is taken. Find the owner with
  `lsof -i :8080` on macOS or `Get-NetTCPConnection -LocalPort 8080` in
  PowerShell before killing anything.
- A Vite smoke check (page renders) is NOT a Power Apps/connector smoke check (data loads
  through the runtime). Verify the actual failing request in the Network tab.

## Publish
Use the npm CLI path first:

```bash
npm run build
pa app push
```

Fallback when the tenant or workflow still relies on PAC:

```bash
npm run build
pac code push
```

Push to a specific solution with the current npm CLI:

```bash
pa solution list --search "<solution-name>" --json
pa app push --solution-id <solution-guid>
```

Build and obtain deployment authorization before push. The npm flag takes a **GUID**, not a display/unique name. Legacy PAC uses `pac code push --solutionName <solution-unique-name>`; retain it only for a confirmed compatibility requirement.

## ALM defaults
- Work in a non-default solution.
- Prefer an explicit solution GUID with npm `--solution-id`; PAC `--solutionName` is a different, legacy selector.
- Use connection references for portable Dev/Test/Prod deployments.
- Use Power Platform Pipelines after the app is solution-aware.

## Multiple data sources
When adding multiple connectors in sequence:
- Inspect generated/config diffs after each add/refresh, then run the relevant build once after the coherent set of integrations is wired. Recheck earlier only to isolate an actual generator failure.
- Do NOT deploy after each connector — deploy once after all connectors are wired.

## Cloud flows (npm CLI only)
Power Automate flows are wired through the npm CLI; `pac code` cannot do this. A flow is
only usable when ALL of these hold — check them in order:

1. the flow is solution-aware;
2. it has a supported Power Apps (manual/`Run`) trigger;
3. the maker has read rights on the flow AND its underlying connections;
4. `list-flows` can discover it;
5. `add-flow` has added the generated types/service/schema + config.

```bash
pa app list-flows --search "<name>" --json
pa app add flow --flow-id <workflow-id>       # generates typed service + schema, edits power.config.json
pa app remove flow --flow-id <workflow-id>
```

After `add-flow`, verify: `power.config.json` (watch for a duplicated `connectionReferences`
key), `.power/schemas/logicflows/`, the generated service/model, `workflowDetails.dependencies`,
missing connection references, then build + Local Play. If the flow definition changes, re-run
`add-flow` with the same id. End users also need runtime permission on the flow. Calling the
generated flow service is covered in [data-access-contract.md](data-access-contract.md).

## Auth and accounts
Browser auth happens on first `init`/`push`, but the CLI is multi-account:

```bash
pa auth login          # add an account (opens browser)
pa auth status         # show cached accounts; the active one is marked
pa auth switch         # choose which cached account other commands run as
pa auth logout         # clears ALL cached accounts — use auth switch to just change active
```

## CLI quick reference
These are canonical grouped operations. Resolve and translate them with the
CLI-resolution section above; never execute a bare unresolved binary. Global
flags commonly include `--json`, `--non-interactive`, and `--no-color`, but the
installed `--help` is authoritative.

```bash
# App lifecycle
pa app init --display-name '<app-name>' --environment-id <env-id>
pa app run --port 8080 --local-app-url <url>
pa app push [--solution-id <id>]                    # build and obtain approval first
pa app list

# Data sources
pa app add data-source --connector <api> [-c <conn-id>] [-d <dataset>] [--table <table>]
pa app refresh data-source [--name <name>]          # regenerate types (NOT delete+re-add)
pa app remove data-source --connector <api> --name <name>
pa app add dataverse-api --api-name <op>
pa app find-dataverse-api --search "<term>"

# Discovery
pa connection list
pa connection list-references --solution-id <solution-guid>
pa app list-environment-variables
pa connection list-datasets --connector <api> -c <conn-id>
pa connection list-tables --connector <api> -c <conn-id> -d <dataset>
pa connection list-procedures -c <conn-id> -d <dataset>

# Connections, flows, auth
pa connection create --connector <api> --display-name "<name>"   # preview; SSO vs browser varies
pa app list-flows --search "<name>" --json
pa app add flow --flow-id <id>
pa app remove flow --flow-id <id>
pa auth login | status | switch | logout
```

> CLI 1.0.1 help consistently specifies `push --solution-id` as a GUID. Older examples used inconsistent solution flags. Check the installed version; never publish just to test a flag.

## App sharing and service-principal publishing

For an already-published app, a maker can grant app access:

```bash
pa app share --principal <user-email-or-entra-object-id> --access play
pa app share --principal <enterprise-application-object-id> --access edit
```

Sharing changes permissions: use the requested principals/access and obtain authorization before executing. For service-principal updates, a maker grants `edit` once using the **Enterprise Application object ID**, not the App Registration object ID or client ID. The service principal also needs environment access and cannot grant itself app access. Do not put sharing in every CI run.

Configure the publish job with `PA_CLI_USE_SP_AUTH=true`, `PA_CLI_SP_CLIENT_ID`, `PA_CLI_SP_TENANT_ID`, and `PA_CLI_SP_CLIENT_SECRET` injected from the CI secret store. Never put the secret in source, logs, prompts, or a browser bundle. `CI=true` also selects service-principal auth for compatibility; prefer the explicit flag. This is CLI publishing auth, separate from Dataverse backend application-user provisioning and end-user connector rights.

After a successful build and deployment authorization:

```bash
pa app push --solution-id <solution-guid> --non-interactive
```

`PA_CLI_*` process variables configure the CLI; they are not Power Platform solution environment variables. Explicit command flags override their corresponding process variables. Verify the target environment from config/flags before publishing.

Sources: [Service-principal publishing](https://learn.microsoft.com/en-us/power-apps/developer/code-apps/how-to/use-service-principal), [CLI environment variables](https://learn.microsoft.com/en-us/power-apps/developer/code-apps/reference/environment-variables).

## App host settings

CLI 1.0.1 supports these grouped commands (verify installed help):

```bash
pa app get-settings --json
pa app set-setting --show-header false
```

Both require `power.config.json`; even `set-setting --help` checks for it in this version. `set-setting` changes local `appSettings`; the player receives the metadata on the next authorized push. It does not change environment CSP or backend/storage CORS. Older package READMEs show flat commands; use only commands exposed by the installed CLI. Reference: [official CLI package](https://www.npmjs.com/package/@microsoft/power-apps-cli).

## What not to do
- Do not start from Next.js or another server-heavy stack unless you already know which parts are purely client-side.
- Do not treat `power.config.json` as application logic.
- Do not publish before local play works.
- Do not assume `pa app push` or `power-apps push` deploys Azure Functions or any other backend resource.
- Do not use Node.js 20 or earlier — v22+ is required.
- Do not call Power Platform or Microsoft 365 directly from the browser. For an
  explicitly approved custom-backend exception, complete the CSP/CORS/security
  preflight in [backend-security.md](backend-security.md#direct-http-calls).
