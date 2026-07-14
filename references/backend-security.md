# Backend and Security Boundaries

## Authentication boundaries
- Power Apps host manages end-user authentication for the app.
- Microsoft Entra is the built-in app-auth path.
- Do not add custom MSAL, OAuth login pages, or SAML login flows for normal Code App auth.
- OAuth can still matter at the connector level.
- SAML belongs in the tenant identity layer, not in the Code App client.

## Direct HTTP calls
Raw `fetch()` from the browser is technically possible because a Code App is still a browser SPA.

Do not treat that as the default architecture.

Use direct browser HTTP only when all of the following are true:
- no secret is required in the client,
- CORS is solved cleanly,
- the endpoint is intended for browser callers,
- the user accepts that the request surface is visible in the browser,
- the data is not sensitive enough to require a stronger boundary.

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
