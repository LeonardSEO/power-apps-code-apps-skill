---
name: power-apps-code-apps
description: Guides building, reviewing, migrating, and deploying Power Apps Code Apps by using the official Vite template, TypeScript, the Power Apps SDK, npm CLI or PAC CLI, connectors, Dataverse, SharePoint, Copilot Studio, and safe backend patterns. Use when creating a new code app, converting an existing SPA, wiring data sources, planning ALM, or checking current platform limits and best practices.
---

# Power Apps Code Apps

Use this skill when the user wants an AI to vibecode a Power Apps Code App correctly instead of treating it like a generic React app or a full-stack Node app.

## Core rules
- Treat a Code App as a browser SPA hosted by Power Apps. App code must end up as browser JavaScript or TypeScript.
- Default stack: Vite + TypeScript + `@microsoft/power-apps`. React is the safest default unless the repo already uses another supported SPA framework.
- The Power Apps host handles end-user authentication and app loading. Do not add custom Entra ID, MSAL, OAuth, or SAML login flows to the app unless the user explicitly wants a separate non-platform auth layer.
- Use generated connector services from `src/generated/...` for Power Platform data access. Do not hand-edit generated files.
- Keep authoritative rules out of the client. Use Dataverse server-side extensibility or an external backend behind a custom connector.

## Workflow
1. Decide whether the task is:
   - new app bootstrap,
   - migration of an existing SPA,
   - data source wiring,
   - backend or security design,
   - deploy or ALM work,
   - or platform clarification.
2. Read the relevant reference file before answering:
   - Bootstrap, updates, local play, deploy: [references/runbook.md](references/runbook.md)
   - Connectors, SharePoint, Dataverse, Copilot Studio: [references/data-integrations.md](references/data-integrations.md)
   - Auth boundaries, backend patterns, Azure Functions, custom connectors: [references/backend-security.md](references/backend-security.md)
   - Limits, gotchas, and official search URLs: [references/limitations-search.md](references/limitations-search.md)
3. Implement or advise in small steps. Prefer one feature, bugfix, or design decision at a time.
4. Verify with the lightest relevant check: `npm run dev`, `npm run build`, `npx power-apps push`, or a focused PAC command.

## Response contract
When doing implementation or giving a vibecoder prompt, keep the answer structured:
- Goal: one sentence
- Files or surfaces touched
- Exact commands
- Risks or platform limits that matter
- What the user should verify next

## Default posture
- Prefer official Microsoft Learn guidance over memory.
- Treat the bundled PDF as a snapshot. If a feature is new or unclear, verify it from the official URLs in [references/limitations-search.md](references/limitations-search.md).
- Correct bad assumptions explicitly. For example, direct `fetch()` from the browser is technically possible, but the secure platform-native pattern is usually connector-first, often with Azure Functions or another backend behind API Management and a custom connector.
