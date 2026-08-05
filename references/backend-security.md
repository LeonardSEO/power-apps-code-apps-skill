# Backend and Security Boundaries

## Authentication boundaries
- Power Apps host manages end-user authentication for the app.
- Microsoft Entra is the built-in app-auth path.
- Do not add custom MSAL, OAuth login pages, or SAML login flows for normal Code App auth.
- OAuth can still matter at the connector level.
- SAML belongs in the tenant identity layer, not in the Code App client.

## Direct HTTP calls
Raw `fetch()` from the browser is technically possible because a Code App is still a browser SPA.

Do not treat that as the default architecture. Dataverse, Microsoft Graph, Microsoft 365, Azure management APIs, and services with a supported Power Platform connector must use generated connector services. A browser call is not a substitute for those connectors.

Use direct browser HTTP to a custom backend only when all of the following are true and the decision is documented:
- no secret is required in the client,
- the endpoint is intended for browser callers,
- the user accepts that the request surface is visible in the browser,
- the data and operation do not require a stronger trust boundary,
- authorization is enforced by the backend rather than trusted client state,
- Power Apps Code Apps CSP permits the destination,
- backend CORS permits the deployed Code App origin,
- storage CORS is configured separately when the browser uploads directly to storage,
- tenant governance and DLP policy permit the architecture,
- Local Play and the deployed Power Apps host are both tested.

In Code Apps, “CORS is solved” is usually not enough by itself. Validate all browser-facing layers:
- Power Apps Code Apps CSP,
- backend CORS,
- and any storage CORS used for direct uploads.

## Recommended backend pattern
For real backend logic, protected APIs, or sensitive evaluation:

```text
Code App
  -> custom connector
  -> API Management
  -> Azure Function or other backend
  -> optional Dataverse / SharePoint / external systems
```

This gives you:
- no secrets in the client bundle,
- reusable connector contracts,
- ALM-friendly solution components,
- better governance and monitoring,
- a place for authoritative business rules.

## Azure Functions
Use Azure Functions or another backend when you need:
- secret handling,
- server-side validation,
- business rules that users must not bypass,
- document or file processing,
- scheduled or compute-heavy work,
- baseline or policy logic you do not want bundled into the client.

Important:
- `power-apps push` does not build or deploy Azure Functions.
- `pa app push` does not build or deploy Azure Functions.
- Backend deployment is a separate pipeline or release step.

## Dataverse server-side logic
Use Dataverse extensibility when you want logic near Dataverse data:
- plug-ins,
- Custom APIs,
- other server-side Dataverse capabilities.

That is the right place for enforcement, not the client.

## What not to do
- Do not hardcode API keys, bearer tokens, connection strings, tenant secrets, or certificates in the client.
- Do not put authoritative security checks only in React state or client validation.
- Do not edit `src/generated` manually.
- Do not store sensitive business rules in the bundle if the requirement says users must not be able to inspect them.
- Do not assume that because a call works in Local Play, it is acceptable for enterprise deployment.
- Do not weaken CSP with broad wildcards merely to make a browser call work.
