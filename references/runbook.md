# Power Apps Code Apps Runbook

## Default stack
- Vite SPA
- TypeScript
- `@microsoft/power-apps`
- npm CLI with grouped `pa` preferred and flat `power-apps` fallback
- PAC CLI for auth, data-source operations, and compatibility paths

## Preflight
Before starting, collect:
- Node.js 22+ installed (`node --version`)
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

In this template, prefer the cross-platform `make code-*` targets; the probes
above are for standalone skill use and troubleshooting.

- Prefer the project-local shim. A global official binary is an existing-tool
  fallback, not permission to install globally.
- If `PA_KIND=none`, run the project's existing `npm install` and re-probe. Ask
  before any global install.
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
| Connections | `pa connection list` | `power-apps list-connections` |
| Create connection | `pa connection create` | `power-apps create-connection` |
| Datasets/tables | `pa connector list-datasets` / `pa connector list-tables` | `power-apps list-datasets` / `power-apps list-tables` |
| Auth | `pa auth login/status/switch/logout` | `power-apps login/auth-status/auth-switch/logout` |

Renamed selector flags:

| Meaning | Grouped `pa` | Flat `power-apps` |
|---|---|---|
| Connector/API | `--connector` | `--api-id` / `-a` |
| Table/resource | `--table` | `--resource-name` / `-t` |
| Data-source name | `--name` | `--data-source-name` / `-n` |
| Connection reference | `--connection-ref` | `--connection-ref` / `-cr` |

Connection (`-c`), dataset (`-d`), environment (`-e`) and init display-name
flags are unchanged. The meaning of `-n` is command-specific; prefer the long
form in generated instructions.

## New app bootstrap
Use the official Microsoft template first.

```bash
npx degit github:microsoft/PowerAppsCodeApps/templates/vite my-app --force
cd my-app
npm install
pa app init --display-name "My Code App" --environment-id <environment-id>
npm run dev
```

Notes:
- Use `--force` with degit to overwrite if the directory already has files.
- Starting with `@microsoft/power-apps` v1.0.4, the npm CLI is the preferred path for `init`, `run`, and `push`.
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
Local Play needs the Vite frontend and Power Apps configuration together. The
current grouped CLI starts the package's `dev` script itself:

```bash
pa app run --port 8080 --local-app-url http://localhost:5173
```

- `--port` is the configuration runtime; `--local-app-url` points at the Vite
  dev server — two different ports. Current `pa app run` starts the configured
  package `dev` script; do not start a second Vite process.
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

Push to a specific solution:

```bash
pac code push --solutionName <solutionName>
```

## ALM defaults
- Work in a non-default solution.
- Prefer a preferred solution or target `--solutionName`.
- Use connection references for portable Dev/Test/Prod deployments.
- Use Power Platform Pipelines after the app is solution-aware.

## Multiple data sources
When adding multiple connectors in sequence:
- Run `npm run build` after each resolved data-source add/refresh operation to catch generated-contract errors early.
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
pa connection list-references
pa app list-environment-variables
pa connector list-datasets --connector <api> -c <conn-id>
pa connector list-tables --connector <api> -c <conn-id> -d <dataset>
pa connector list-procedures --connector <api> -c <conn-id> -d <dataset>

# Connections, flows, auth
pa connection create --connector <api> --display-name "<name>"   # preview; SSO vs browser varies
pa app list-flows --search "<name>" --json
pa app add flow --flow-id <id>
pa app remove flow --flow-id <id>
pa auth login | status | switch | logout
```

> Live CLI help can itself be inconsistent — e.g. `push --help` shows `--solution-id` while
> the description/example use `--solution-name`. Verify a flag against the installed version;
> never run a production `push` just to "test" a flag.

## What not to do
- Do not start from Next.js or another server-heavy stack unless you already know which parts are purely client-side.
- Do not treat `power.config.json` as application logic.
- Do not publish before local play works.
- Do not assume `pa app push` or `power-apps push` deploys Azure Functions or any other backend resource.
- Do not use Node.js 20 or earlier — v22+ is required.
- Do not call Power Platform or Microsoft 365 directly from the browser. For an
  explicitly approved custom-backend exception, complete the CSP/CORS/security
  preflight in [backend-security.md](backend-security.md#direct-http-calls).
