# Power Apps Code Apps Skill

A cross-compatible agent skill for building and reviewing **Power Apps Code Apps** with the right stack, boundaries, deployment flow, and data patterns.

This skill is written as a **single-skill portable bundle** so it can be shared with:
- Codex
- Claude Code
- other agents that support the open `SKILL.md` format

It is designed for AI vibecoding workflows where the model needs to understand:
- how Power Apps Code Apps actually work,
- which stack to use,
- how to bootstrap and deploy them,
- how to connect data sources safely,
- where the platform boundaries are,
- and when to switch to Azure Functions, Dataverse extensibility, or custom connectors.

## What this skill covers
- Official bootstrap flow with the Microsoft Vite template
- TypeScript, Vite, and `@microsoft/power-apps` defaults
- npm CLI and PAC CLI usage
- Local Play and browser gotchas
- CSP, CORS, white-screen, caching, and runtime-state troubleshooting
- Dataverse integration
- Dataverse provisioning, solution import, and Web API metadata fallback
- SharePoint boundaries
- Copilot Studio agent integration
- Connection references and ALM
- Backend patterns with Azure Functions, API Management, and custom connectors
- Platform limitations and what not to do

## Repository structure
```text
power-apps-code-apps-skill/
├── README.md
├── LICENSE
├── SKILL.md
└── references/
    ├── backend-security.md
    ├── dataverse-provisioning.md
    ├── data-integrations.md
    ├── limitations-search.md
    ├── runbook.md
    └── troubleshooting.md
```

## Install

### Shared install via Skills CLI
After publishing this folder as a GitHub repository, install it with:

```bash
npx skills add LeonardSEO/power-apps-code-apps-skill
```

If your installed Skills CLI supports agent targeting, use its help output to see the current flags:

```bash
npx skills add --help
```

The repository itself is intentionally kept tool-neutral so the installer layer can decide where to place it.

## Manual install

### Claude Code
Copy this repository to one of these locations:

```text
.claude/skills/power-apps-code-apps/
~/.claude/skills/power-apps-code-apps/
```

### Codex
Copy this repository to your skills location or wrap it in your preferred Codex distribution layer. The skill itself is intentionally portable and keeps the canonical instructions in the root `SKILL.md`.

## Invoke
Use the skill by name:

```text
/power-apps-code-apps
```

Or reference it from an agent prompt when you want the model to act as a Power Apps Code Apps specialist.

## Design goals
- One canonical `SKILL.md`
- Tool-neutral instructions
- Compatible with both Codex and Claude Code style skill discovery
- Minimal assumptions beyond the shared skill format
- Official Microsoft Learn first, local PDF second

## Authoring notes
This skill intentionally separates:
- the **skill bundle** for sharing,
- from the **personal installed copy** in `~/.codex/skills`.

That keeps the distributable version clean and repo-friendly.

## License
MIT
