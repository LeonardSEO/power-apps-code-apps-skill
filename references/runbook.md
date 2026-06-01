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
npx power-apps init -n "My Code App" -e <environment-id>
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

## Local development
```bash
npm run dev
# or
npx power-apps run
```

Expected result:
- Vite starts
- a `Local Play` URL is printed
- the app can open inside the Power Apps host

## Publish
Use the npm CLI path first:

```bash
npm run build
npx power-apps push
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
- Run `npm run build` after each `npx power-apps add-data-source` to catch errors early.
- Do NOT deploy after each connector — deploy once after all connectors are wired.

## CLI quick reference

```bash
npx power-apps init -n '<app-name>' -e <env-id>          # Initialize, creates power.config.json
npx power-apps push                                        # Deploy (run npm run build first)
npx power-apps run                                         # Start local dev server
npx power-apps add-data-source -a <api> [-c <conn-id>] [-d <dataset>] [-t <table>]
npx power-apps list-connections                            # List available connections
npx power-apps list-datasets -a <api> -c <conn-id>        # List datasets for connector
npx power-apps list-tables -a <api> -c <conn-id> -d <dataset>  # List tables in dataset
npx power-apps logout                                      # Clear cached auth token
```

## What not to do
- Do not start from Next.js or another server-heavy stack unless you already know which parts are purely client-side.
- Do not treat `power.config.json` as application logic.
- Do not publish before local play works.
- Do not assume `npx power-apps push` deploys Azure Functions or any other backend resource.
- Do not use Node.js 20 or earlier — v22+ is required.
- Do not use `fetch()` or `axios` directly — use generated connector services instead.
