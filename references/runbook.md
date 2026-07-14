# Power Apps Code Apps Runbook

## Default stack
- Vite SPA
- TypeScript
- `@microsoft/power-apps`
- npm CLI for the current happy path
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

## New app bootstrap
Use the official Microsoft template first.

```bash
npx degit github:microsoft/PowerAppsCodeApps/templates/vite my-app --force
cd my-app
npm install
power-apps init -n "My Code App" -e <environment-id>
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

## Local Play (two processes)
Local Play is NOT just Vite. It needs the Vite frontend AND the Power Apps connection
runtime running together (that is what `make dev-dataverse`-style scripts wrap):

```bash
npm run dev                                   # 1) Vite frontend (default :5173)
power-apps run \                              # 2) Power Apps connection runtime
  --port 8080 \                               #    connection-runtime port
  --local-app-url http://localhost:5173       #    where Vite serves the app
```

- `--port` is the connection runtime; `--local-app-url` points at the Vite dev server —
  two different ports. `power-apps run` does not itself start your Vite app.
- Let the CLI print the `Local Play` URL and open THAT. Do not hand-build a URL from an
  old app id — that yields `Launch App failed with Http status code of 0`.
- Open it in the **same browser profile** as the Power Platform tenant. If it fails in
  Chrome/Edge, check **Local Network Access** permissions before touching code.
- `EADDRINUSE :::8080` (or 5173) means the port is taken. Find the owner first
  (`lsof -i :8080`) before killing anything.
- A Vite smoke check (page renders) is NOT a Power Apps/connector smoke check (data loads
  through the runtime). Verify the actual failing request in the Network tab.

## Publish
Use the npm CLI path first:

```bash
npm run build
power-apps push
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
- Run `npm run build` after each `power-apps add-data-source` to catch errors early.
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
power-apps list-flows --search "<name>" --json
power-apps add-flow --flow-id <workflow-id>       # generates typed service + schema, edits power.config.json
power-apps remove-flow --flow-id <workflow-id>
```

After `add-flow`, verify: `power.config.json` (watch for a duplicated `connectionReferences`
key), `.power/schemas/logicflows/`, the generated service/model, `workflowDetails.dependencies`,
missing connection references, then build + Local Play. If the flow definition changes, re-run
`add-flow` with the same id. End users also need runtime permission on the flow. Calling the
generated flow service is covered in [data-access-contract.md](data-access-contract.md).

## Auth and accounts
Browser auth happens on first `init`/`push`, but the CLI is multi-account:

```bash
power-apps login          # add an account (opens browser)
power-apps auth-status     # show cached accounts; the active one is marked
power-apps auth-switch     # choose which cached account other commands run as
power-apps logout          # clears ALL cached accounts — use auth-switch to just change active
```

## CLI quick reference
`power-apps` = the project-local binary (`./node_modules/.bin/power-apps`), not bare `npx`
(see SKILL.md). Read live help with `power-apps --help` / `power-apps <verb> --help`
(the bare `help` verb is unsupported; `init --help` needs an empty dir). Global flags:
`--json`, `--non-interactive`, `--no-color`.

```bash
# App lifecycle
power-apps init -n '<app-name>' -e <env-id>        # creates power.config.json (needs empty dir)
power-apps run --port 8080 --local-app-url <url>   # connection runtime for Local Play
power-apps push [--solution-name <name>]           # deploy (run npm run build first)
power-apps list-codeapps

# Data sources
power-apps add-data-source -a <api> [-c <conn-id>] [-d <dataset>] [-t <table>]
power-apps refresh-data-source [--data-source-name <name>]   # regenerate types (NOT delete+re-add)
power-apps delete-data-source --api-id <api> --data-source-name <name>
power-apps add-dataverse-api --api-name <op>       # Custom API / action (flag is --api-name)
power-apps find-dataverse-api --search "<term>"

# Discovery
power-apps list-connections
power-apps list-connection-references
power-apps list-environment-variables
power-apps list-datasets -a <api> -c <conn-id>
power-apps list-tables -a <api> -c <conn-id> -d <dataset>
power-apps list-sqlStoredProcedures -a <api> -c <conn-id> -d <dataset>

# Connections, flows, auth
power-apps create-connection --api-id <api> --display-name "<name>"   # preview; SSO vs browser varies
power-apps list-flows --search "<name>" --json
power-apps add-flow --flow-id <id>
power-apps remove-flow --flow-id <id>
power-apps login | auth-status | auth-switch | logout
```

> Live CLI help can itself be inconsistent — e.g. `push --help` shows `--solution-id` while
> the description/example use `--solution-name`. Verify a flag against the installed version;
> never run a production `push` just to "test" a flag.

## What not to do
- Do not start from Next.js or another server-heavy stack unless you already know which parts are purely client-side.
- Do not treat `power.config.json` as application logic.
- Do not publish before local play works.
- Do not assume `power-apps push` deploys Azure Functions or any other backend resource.
- Do not use Node.js 20 or earlier — v22+ is required.
- Do not use `fetch()` or `axios` directly — use generated connector services instead.
