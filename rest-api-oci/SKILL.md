---
name: rest-api-oci
description: Use when designing, reviewing, or implementing REST APIs and OpenAPI/Swagger specs for consistent resource-oriented API design. Applies to CRUDL resource APIs, operation names, URI and HTTP method choices, request/response models, list pagination/filtering/sorting, errors, lifecycle state, optimistic concurrency, idempotency, versioning, naming, and production API review readiness. The guidance is generic REST guidance derived from OCI API consistency best practices; apply OCI-specific conventions only when the target API uses or wants OCI-style contracts.
---

# REST API Design

Use this skill to design or review REST APIs using a strict, production-oriented checklist derived from OCI API Consistency Guidelines.

## Workflow

1. Identify the resource model, whether resources have stable identifiers, and the intended operations.
2. Prefer standard CRUDL shape before introducing custom actions or batch operations.
3. Check operation IDs, HTTP methods, URI paths, request body schemas, success responses, errors, pagination, lifecycle state, and naming.
4. For detailed rules and checklists, read `references/rest-api-guidelines.md`.
5. Adapt organization-specific terms such as OCID, work request, Terraform, SDK, CLI, and `opc-*` headers to the target platform's equivalents.
6. When reviewing an API, report concrete design issues first, then suggested corrections and any exception-worthy tradeoffs.

## Core Rules

- Define public APIs in OpenAPI/Swagger and treat generated clients, documentation, and examples as part of the contract.
- Choose an explicit versioning strategy. A version segment such as `/v1` is common; date-based versions such as `/20160918` are a stricter OCI-style option.
- Keep the base path focused on API/version scope; use top-level plural resource paths like `/{version}/widgets/{widgetId}`.
- Use paths and query strings only to identify the target resource set. Use HTTP method and request body to define the action.
- Use standard methods: `POST` create, `GET` list/get, `PUT` replace or documented merge update, `PATCH` partial update, `DELETE` delete, `HEAD` metadata/existence checks.
- Model non-CRUD actions as `POST` on `/actions/<actionName>` under the specific resource, or under top-level `/actions` only when no stable resource is affected.
- Use operation IDs in the form `<Verb><ResourceType>` or `List<PluralResourceType>`.
- Prefer complete CRUDL coverage for first-class resources when the resource lifecycle supports it.
- Use `application/json` for structured request and response bodies unless the operation is explicitly raw bytes/content.

## Design Biases

- Preserve service-local consistency when existing APIs already diverge, but use this guidance for new services and new patterns.
- Do not expose internal architecture in tags, paths, clients, or resource boundaries.
- Avoid subresources when a child can be a top-level resource with a parent ID field.
- Avoid custom lifecycle states, ad hoc error codes, unbounded arrays/maps, naked array/map models, aliases, and nested model definitions.
- Treat generated SDKs, CLIs, infrastructure-as-code providers, and documentation behavior as part of the API contract when those artifacts exist.
