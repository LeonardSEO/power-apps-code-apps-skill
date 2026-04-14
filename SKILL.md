---
name: power-apps-code-apps
description: Guides building, debugging, migrating, and deploying Power Apps Code Apps by using the official Vite template, TypeScript, the Power Apps SDK, npm CLI or PAC CLI, connectors, Dataverse, SharePoint, Copilot Studio, and safe backend patterns. Use when creating a new code app, converting an existing SPA, wiring data sources, provisioning Dataverse-backed storage, debugging white screens or fetch failures, planning ALM, or checking current platform limits and best practices.
---

# Power Apps Code Apps

Use this skill when the user wants an AI to vibecode a Power Apps Code App correctly instead of treating it like a generic React app or a full-stack Node app.

## Core rules
- Treat a Code App as a browser SPA hosted by Power Apps. App code must end up as browser JavaScript or TypeScript.
- Default stack: Vite + TypeScript + `@microsoft/power-apps`. React is the safest default unless the repo already uses another supported SPA framework.
- The Power Apps host handles end-user authentication and app loading. Do not add custom Entra ID, MSAL, OAuth, or SAML login flows to the app unless the user explicitly wants a separate non-platform auth layer.
- Use generated connector services from `src/generated/...` for Power Platform data access. Do not hand-edit generated files.
- Keep authoritative rules out of the client. Use Dataverse server-side extensibility or an external backend behind a custom connector.
- Split the release path explicitly: Code App frontend push, backend publish, and Power Platform or Azure settings are separate concerns.
- Distinguish target architecture from runtime truth. Having Dataverse enabled, a repository class in code, or a schema document does not prove that runtime data is actually stored in Dataverse.

## Workflow
1. Decide whether the task is:
   - new app bootstrap,
   - migration of an existing SPA,
   - data source wiring,
   - Dataverse provisioning and schema setup,
   - troubleshooting of CSP, CORS, runtime state, or white screens,
   - backend or security design,
   - deploy or ALM work,
   - or platform clarification.
2. Read the relevant reference file before answering:
   - Bootstrap, updates, local play, deploy: [references/runbook.md](references/runbook.md)
   - Connectors, SharePoint, Dataverse, Copilot Studio: [references/data-integrations.md](references/data-integrations.md)
   - Dataverse environment, schema, CLI, Web API, and generated services: [references/dataverse-provisioning.md](references/dataverse-provisioning.md)
   - Preflight, release split, cache/debug, and symptom-to-fix guidance: [references/troubleshooting.md](references/troubleshooting.md)
   - Auth boundaries, backend patterns, Azure Functions, custom connectors: [references/backend-security.md](references/backend-security.md)
   - Limits, gotchas, and official search URLs: [references/limitations-search.md](references/limitations-search.md)
3. Before implementation, gather the preflight facts from [references/troubleshooting.md](references/troubleshooting.md): environment id, Dataverse org URL, backend host, blob host, auth model, direct fetch versus connector choice, and runtime storage truth.
4. Implement or advise in small steps. Prefer one feature, bugfix, or design decision at a time.
5. When Dataverse is involved, prefer solution import for schema, `pac code add-data-source` for code-app connectivity, and Web API metadata only as an advanced fallback.
6. Verify with the lightest relevant check: `npm run dev`, `npm run build`, `npx power-apps push`, a focused PAC command, or a runtime smoke test against the actual failing request.

## Response contract
When doing implementation or giving a vibecoder prompt, keep the answer structured:
- Goal: one sentence
- Files or surfaces touched
- Preflight assumptions
- Exact commands
- Risks or platform limits that matter
- What the user should verify next

## Default posture
- Prefer official Microsoft Learn guidance over memory.
- Treat the bundled PDF as a snapshot. If a feature is new or unclear, verify it from the official URLs in [references/limitations-search.md](references/limitations-search.md).
- Correct bad assumptions explicitly. For example, direct `fetch()` from the browser is technically possible, but the secure platform-native pattern is usually connector-first, often with Azure Functions or another backend behind API Management and a custom connector.
- When a user says “use Dataverse,” clarify whether they mean environment capability, schema existence, generated data sources, or actual runtime persistence.
