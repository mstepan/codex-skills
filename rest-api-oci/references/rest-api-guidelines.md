# REST API Guidelines Reference

Source: extracted and generalized from OCI API Consistency Guidelines.

Use this reference as generic REST API design guidance. Many rules came from OCI's stricter public cloud API contract; when the target API is not OCI-style, translate platform-specific names and mechanisms to the closest local equivalent:

- `OCID` means a stable resource identifier.
- `opc-request-id` means a trace/request correlation ID.
- `opc-retry-token` means an idempotency key.
- `opc-next-page` means an opaque next-page token.
- Work requests mean asynchronous operation/job resources.
- OCI SDK/CLI/Terraform constraints mean generated clients, command-line tools, infrastructure-as-code providers, or other downstream integrations.
- OCI-specific requirements such as date-based version paths, Swagger 2.x, `x-obmcs-*` extensions, and exact lifecycle enums are strict baselines, not universal requirements.

## Review Checklist

- An OpenAPI/Swagger spec or equivalent contract exists and can be validated.
- API versioning is explicit and applies consistently to the service, not separately per resource.
- Resource paths are top-level plural nouns unless there is a clear exception.
- Operation IDs match method semantics and resource names.
- CRUDL operations use standard HTTP methods and response shapes.
- Create/update/delete operations declare idempotency, concurrency, and async behavior consistently.
- List/analytics operations use wrapped collections, filters, sorting, and pagination-compatible headers/parameters.
- Errors declare expected HTTP responses and use a bounded, documented status/error-code taxonomy.
- Models avoid unbounded structures, sensitive fields, unstable names, and breaking SDK naming changes.
- Lifecycle states use a small documented state set and are not caller-settable.
- Tags/operation groups, ETags, request IDs, retry tokens, dry-run behavior, and default values are modeled explicitly where applicable.

## Versioning and Breaking Changes

- Version APIs explicitly in a predictable place, commonly a path segment such as `/v1`; date-based path versions such as `/20160918` are an OCI-style stricter option.
- The version should represent the whole service version. If a breaking revision requires a new version, support that version consistently across service resources.
- A breaking change is any schema, URI, response, or behavior change that makes previously working customer code fail without customer action.
- Common breaking changes include changing an optional parameter default, changing sort defaults, adding pagination to a formerly unpaginated one-page operation, making a request field required, making a response field optional, changing constraints in a less-compatible direction, adding mutual exclusion constraints, changing retry/circuit-breaker settings, or making a synchronous operation asynchronous.
- Adding a new enum value should usually be treated as compatible only when generated clients and consumers are designed to tolerate unknown values.
- Do not change schema/reference names casually; this can break generated clients in some languages.

## URIs and HTTP Methods

- Path and query components identify the resource or collection being acted on. They should not encode what action to perform.
- `Create<ResourceType>`: `POST` to the parent collection.
- `List<PluralResourceType>`: `GET` on the parent collection.
- `Get<ResourceType>`: `GET` on a single-resource path.
- `Update<ResourceType>`: `PUT` when the API defines replace or documented merge-update semantics.
- `Patch<ResourceType>`: `PATCH`, accepting a documented partial-update format such as JSON Patch, merge patch, or OCI-style JSON List Patch.
- `Delete<ResourceType>`: `DELETE` on a single-resource path.
- `Head<ResourceType>`: `HEAD`.
- Other operations: `POST` to `/actions/<operationName>`.
- First-class resources should usually provide Create, Get, Update, Delete, and List.

## Operation IDs

- Use `<Verb><ResourceType>` for single-resource operations.
- Use `List<PluralResourceType>` for same-type collections.
- Use `Summarize<PluralEntityType>` for GET analytics and `RequestSummarized<PluralEntityType>` for POST analytics.
- Use `Attach<ResourceType>` and `Detach<ResourceType>` for association operations.
- Standard action verbs include `Change`, `CascadingDelete`, `Schedule`, `Cancel`, `Add`, and `Remove` for the specific approved patterns.
- Avoid legacy verbs like `Terminate`, `Provision`, generic `Add`, or generic `Remove` when a standard CRUD/action pattern exists.

## Actions

- Use `POST /<resources>/{resourceId}/actions/<actionName>` for actions scoped to a resource.
- Select the action solely from the path, not query parameters or headers.
- Actions that primarily modify a resource should accept `If-Match` and compare it to the resource ETag.
- A synchronous action returning the updated resource uses `200`, `Content-Location`, and the updated resource body.
- A fully asynchronous action uses `202`, a job/work-request identifier header or body field, and no immediate resource body.
- Do not use actions to add/remove items from arrays in a resource. Use `Patch<ResourceType>` with JSON List Patch.

## Headers

- Keep total request headers below 16 KiB where possible.
- Every operation should include a request/correlation ID in responses; OCI uses `opc-request-id`.
- Create operations should accept an idempotency key to support retryable creates; OCI uses `opc-retry-token`.
- Operations modifying a preexisting resource should generally support `If-Match`.
- GET and HEAD may support `If-None-Match`.
- `retry-after` may be returned with retryable response statuses and should usually be a non-negative integer number of seconds.
- Unknown reserved vendor headers should be rejected with `400` and a documented invalid-parameter error; OCI specifically rejects unknown `opc-*` headers.

## Success Responses

- Every operation declares explicit success responses.
- Synchronous create: `200` with `<ResourceType>` body. Asynchronous create: `201` or `202` depending on the async pattern. Dry-run create may use `204` or `202`.
- Update: `200` with `<ResourceType>` body or `202` for async.
- List: `200` with `<ResourceType>Collection`, containing `items: <ResourceType>Summary[]`.
- Get: `200` with `<ResourceType>`.
- Delete: `204` or `202`; no response body.
- Multiple 2xx responses are allowed, but generated client tooling may have constraints. Higher-numbered 2xx responses should not introduce response headers absent from the lowest 2xx response unless tooling supports it.

## Errors

- Errors should consist of an HTTP status, a stable application error code, and a short message.
- Use only documented HTTP status codes and application error codes. New codes should go through the API governance/review process.
- Every operation must declare at least `401`, `404`, `429`, and `500`.
- Declare HTTP statuses in the OpenAPI/Swagger spec. Document operation-specific application error codes in user documentation or the error schema.
- Response descriptions should match the well-known HTTP reason phrase, such as `Unauthorized` or `Not Found`.
- Sanitize messages. Do not return stack traces or raw framework/deserialization errors.
- Unknown parameters should return `400` with `InvalidParameter`.
- Missing/invalid list parameters generally return `400`; authorization failures for list inspection return `404 NotAuthorizedOrNotFound`.

## Naming

- Operation IDs and model names use PascalCase, for example `CreateLoadBalancer`.
- Paths, parameters, and JSON fields use camelCase, for example `/loadBalancers/{loadBalancerId}`.
- Use `id` and `<resourceType>Id`, not `ID`.
- Capitalize only the first letter of embedded acronyms, except standardized unit abbreviations.
- Boolean fields should usually use positive auxiliary-verb names: `isEnabled`, `hasFoo`, `canDelete`.
- Consider enums instead of booleans when future states are plausible.
- Resource input schemas use `<Verb><ResourceType>Details`.
- The full output schema returned by Get/Create/Update is `<ResourceType>`.
- Fields referencing other resources should state the referenced property, such as `vcnId`, `shapeName`, `objectNames`.
- Numeric fields with unclear units should include units in the field name, such as `latencyInMilliseconds`.
- Enum values use uppercase US English identifiers with underscores, such as `IN_PROGRESS`, except values that identify resource fields should match field names exactly.

## Resource Models

- Fields that can be specified in Create or Update should appear in Get responses.
- All operations returning `<ResourceType>` should return the same fields.
- Response bodies should generally include all fields, even when values are null.
- Empty string should be interpreted as null. Null is a defined value. Missing key is undefined.
- Arrays in request/response models must specify `maxItems`; if non-empty, specify `minItems`.
- Unpaginated arrays/maps must be bounded with `maxItems`/`maxProperties`.
- A resource response should be no more than 500 KiB, especially behind SPLAT.
- Avoid arrays or maps that can grow too large in a resource.
- Do not define naked array/map models, aliases, or nested model definitions.

## Lifecycle State

- Resources should expose `lifecycleState`.
- Allowed normal-resource states are: `CREATING`, `ACTIVE`, `INACTIVE`, `UPDATING`, `NEEDS_ATTENTION`, `DELETE_SCHEDULED`, `DELETING`, `DELETED`, and `FAILED`.
- Do not invent custom lifecycle states. Ask for guideline/API review if a general state seems missing.
- Once a resource reaches a terminal state, prohibit client updates other than deletion.
- Do not put `lifecycleState` in Create or Update details; callers should not set it directly.
- Use `lifecycleDetails` for substate and explanatory text.
- Define lifecycle states once in the full resource schema, then reuse that definition from summary/filter schemas using the platform's schema-reference mechanism. OCI uses `x-obmcs-enumref`.
- Task-like first-class resources should usually be work requests; if represented as resources, match work request status values.
- Deleted resources should preferably be tombstoned, retained at least 24 hours and ideally 90 days, returned by Get/List as applicable, and not counted as active references.

## Create APIs

- Create must use `POST` and operation ID `Create<ResourceType>`.
- Create should accept an idempotency key.
- Synchronous create returns the resource.
- Async create should return or link to an operation/job resource; OCI work request patterns use `opc-work-request-id`.
- Create details are named `Create<ResourceType>Details`.
- `displayName` should be optional; if omitted, generate `<resourceType><timeCreated>` using UTC basic ISO timestamp at second precision.
- `name` usually means a user-supplied unique identifier and should generally be required.
- Cloning should be modeled as polymorphic create where possible. Use discriminator value `CLONE`, `CloneCreate<Resource>Details`, and `source<Resource>Id`.

## Get and Head APIs

- Get targets one resource and returns `<ResourceType>`.
- Get requires `<ResourceType>_READ`.
- Platform resources available to everyone require special `compartmentId` authorization behavior; reject ambiguous requests with `404 NotAuthorizedOrNotFound`.
- HEAD operation IDs must start with `Head`, must use `HEAD`, have no body parameter, return `200`, no body, and an ETag on success.

## Update and Patch APIs

- `Update<ResourceType>` uses `PUT` only when its replace or merge semantics are clearly documented. OCI uses shallow-merge partial update semantics for this operation name.
- All fields in `Update<ResourceType>Details` must be optional.
- Top-level fields present in the request replace the stored value completely; absent fields remain unchanged.
- If a field is submitted with the same current value, do not reject it.
- Prefer updating all resource fields in one request. If this can time out or requires workflows, return a work request and perform async.
- Use `Patch<ResourceType>` with a documented patch format, such as JSON Patch, merge patch, or OCI JSON List Patch, for precise array/deep-object changes or atomic updates across subresources.
- Use `Put<ResourceType>` only for true create-or-replace semantics.

## Delete APIs

- `Delete<ResourceType>` deletes one resource immediately or starts deletion as soon as possible.
- DELETE must exist even when cascading or scheduled delete actions also exist.
- Cascading delete is an action named `CascadingDelete<Resource>`, returns a work request ID, returns `202`, and deletes child resources before parents.
- Scheduled delete is an action named `Schedule<Resource>Deletion`; cancel is `Cancel<Resource>Deletion`, synchronous, returning `204`.
- Scheduled deletion should use `DELETE_SCHEDULED` while cancelable, then `DELETING` once deletion cannot be canceled.
- Scheduled cascading delete uses `ScheduleCascading<Resource>Deletion`; cancel uses `CancelScheduledCascading<Resource>Deletion`.

## List APIs

- List APIs use `GET` and target collections.
- Prefer wrapped collections: `<ResourceType>Collection` with required `items: <ResourceType>Summary[]`.
- Naked array list responses are deprecated and must not be used for new services unless preserving existing service-wide style.
- `<ResourceType>Summary` should be a subset of `<ResourceType>` and omit sensitive or overly detailed fields.
- List requires `<ResourceType>_INSPECT`, checked at request scope, not against only the currently matching rows.
- If the resource has these fields, List/Get should return: `lifecycleState`, `name`, `displayName`, `timeCreated`, `id`, `compartmentId` or parent ID, `availabilityDomain`, `self`, tags, and `clusterPlacementGroupId`.
- Empty result sets return `200` with an empty collection, not `404`.
- Limit list response size to about 500 KiB; returning fewer than requested is acceptable.

## Filtering and Sorting

- List and analytics operations should require at least one query parameter that bounds matching resources to a compartment or parent-compartment scope.
- Resources with stable identifiers should support filtering by `id` for specific-resource listing.
- List APIs should support optional `lifecycleState` and `name`/`displayName` exact-match filters when applicable.
- Filter parameter names should match model fields, with operator suffixes when not exact match, for example `yearGreaterThanOrEqualTo`.
- Different fields combine as AND. Repeated same-field exact filters combine as OR. Repeated negated filters combine as AND.
- Use `collectionFormat: multi` for repeatable query parameters.
- Do not filter list responses down to what the caller can inspect; authorize the request scope or fail.
- Default list sort should be `timeCreated` newest first.
- Support `sortBy` enum values matching resource property names and `sortOrder` values `ASC` and `DESC`.
- Accept `sortBy`, `sortOrder`, and enum filters case-insensitively.
- Complex filtering/sorting usually belongs in Resource Query Service or analytics, not regular List APIs.

## Pagination

- All List and Analytics APIs should implement pagination. Even if not implemented yet, define the spec to support future pagination.
- Declare an optional next-page token response field/header and a `page` query parameter on List/Analytics operations; OCI uses `opc-next-page`.
- Page tokens are opaque; clients must not parse or modify them.
- Prefer live pagination semantics over snapshot semantics.
- The final page should omit the next-page token.
- `page` must be a previously returned token. Invalid tokens return `400 BadRequest`.
- Page tokens should encode all original query parameters. If `page` is supplied with different query parameters, return `400 BadRequest`.
- Paginated operations should accept positive-integer `limit`. Invalid limits return `400 BadRequest`.
- An empty page does not mean the end; absence of the next-page token means the end.

## Analytics APIs

- Analytics APIs aggregate summary information and should usually support GET; support POST when filters/sorts can exceed URL length.
- If POST analytics is supported, still pass `limit` and `page` as query parameters.
- Return an object with `items`, each item being a flat `<EntityType>Aggregation`.
- Include scope information such as time interval and dimensions so each aggregation is meaningful alone.
- When returning multiple metrics, add `metricName` and include units in metric names where applicable.
- Authorization should be all-or-nothing unless an explicit opt-in parameter such as `accessLevel=ACCESSIBLE` is intentionally supported.

## Optimistic Concurrency

- Responses containing one specific resource from Create/Get/Update should include a strong ETag when the resource can be updated or deleted.
- If Get returns ETag, synchronous Create/Update returning the resource must also return ETag.
- Operations modifying a single preexisting resource should accept optional `If-Match` and return `412` when it does not match.
- GET may accept optional `If-Match` and fail with `412` on mismatch.
- GET and HEAD may accept `If-None-Match` and return `304` with empty content on match.

## Idempotency, Retries, and Dry Run

- Create should support an idempotency key for retryable creates.
- Delete idempotency: if soft-deleted/tombstoned resource still exists, repeated delete should return `204`; if fully gone, return `404`.
- Generated-client default retries and circuit breakers are part of API behavior; changing settings can be breaking.
- Operations may accept a dry-run flag/header; OCI uses `opc-dry-run: true`.
- For dry-run operations that normally return a resource, return `204` and no body.
- For dry-run operations that normally return `202`, continue returning `202` with a dry-run work request ID.
- Fail dry-run requests synchronously when possible rather than creating a dry-run work request for known failures.

## Tags

- Every operation must have exactly one OpenAPI `tags` value.
- The tag or operation group often determines generated client grouping and endpoint. All operations in the same generated client should share endpoint semantics.
- Use customer-oriented grouping; avoid exposing microservice boundaries.
- Tag values should be a single descriptive word or camelCase phrase, singular preferred.
- Resource tags use `freeformTags`, `definedTags`, and `systemTags`; if a resource has tags, both full and summary models must include them.

## Parameters and Defaults

- Allowed parameter locations are `path`, `query`, `header`, and `body`; `formData` is not allowed.
- Array query parameters must use `collectionFormat: multi`.
- Most optional parameters need either `default` or `x-default-description`, but not both.
- Use `default` only for static non-null defaults.
- Use `x-default-description` for null, dynamic, or context-preserving defaults.
- Changing an optional parameter default is breaking.
- Do not duplicate redundant parameter names across path/query/header/body. If two values are different, give one a semantic prefix or suffix, such as `targetCompartmentId`.
- Prefer secret-resource references over direct password fields. If accepting a secret-like input, use `format: password` or the platform's equivalent.

## Polymorphism

- For polymorphism, prefer a discriminator on a base schema and derived schemas using the OpenAPI mechanism supported by the target toolchain. OCI's Swagger 2.x-compatible pattern uses a discriminator and `allOf`.
- The base schema has the discriminator field and requires it.
- Existing non-polymorphic models cannot be made directly polymorphic without a breaking change. Use migration-compatible wrapper patterns when needed.
- If an API can reasonably expect future variants, make it polymorphic from the start.

## Data Consistency

- Store and return user-provided resource field values exactly; do not trim, normalize, or rewrite values. Reject unacceptable input instead.
- Compare static system values case-insensitively where appropriate, but use locale-stable ASCII handling.
- If identifiers are case-insensitive, preserve input casing when storing/reporting unless the contract explicitly normalizes them.
- Region/location fields should use public stable identifiers, not internal short codes. For example, OCI uses `us-phoenix-1`, not `PHX`.
- Operations must atomically update the underlying data store or atomically record that async work has started, usually via lifecycle state.

## Time, Duration, and Locale

- Point-in-time fields should be named `time<Action>` in past tense, for example `timeCreated`.
- Use `type: string`, `format: date-time`, accept RFC 3339 input with timezone offset, and return RFC 3339 values with `Z`.
- Correctly interpret up to 9 fractional-second digits or document/reject unsupported precision.
- Recurring times should use RFC 5545 recurrence syntax or an explicitly documented equivalent; OCI marks this with `format: x-obmcs-recurring-time`.
- Durations should use ISO 8601 extended duration strings or numeric `<property>In<unit>` fields; OCI marks duration strings with `format: x-obmcs-duration`.
- Time zone fields are named `<Prefix>TimeZone`, use IANA time zone names, and need an update SLA for timezone database changes.
- Language fields should be named `language` or `<Prefix>Language` when scope is not the whole resource.
- Currency fields should use ISO 4217 currency codes unless the API review process accepts additional values.

## Sensitive and Arbitrary Data

- URL paths must not contain sensitive information.
- Headers and query parameters should not contain sensitive information unless unavoidable.
- If Create must return sensitive one-time data, return `<ResourceType>CreateResult` instead of the normal resource model.
- Arbitrary nested customer JSON fields use `type: object`, remain bounded, and are still subject to API review.
- Arbitrary non-JSON payload fields use `type: string` with an explicit encoding such as base64.
- APIs returning or receiving raw bytes/content follow the dedicated raw-content patterns rather than normal JSON model patterns.

## Subresources and Resource Boundaries

- Prefer top-level resources: `/{version}/childResources/{childResourceId}` with a `parentResourceId` field.
- Use subresource paths only for special cases such as singletons or small extended details that are too large for Get responses.
- Subresources remain subject to resource size, auth, and consistency rules.

## Batch APIs

- Batch create is not approved.
- Batch update/patch may be an action on a resource collection.
- Batch update/patch may combine existing individual Update/Patch operations only; it must not replace those individual operations.
- Batch update/patch must not perform create, delete, get, list, or arbitrary actions.
- Authorize per target resource/subresource. Individual failures should be represented without stopping the whole batch where possible.
- Operation ID is `BatchUpdate<ResourcePlural>`.
- Input model is `BatchUpdate<ResourcePlural>Details` with `operations`.
- Use a polymorphic operation base with discriminator `operation`.

## Generated Tooling Compatibility

- Infrastructure-as-code and generated-client tooling are sensitive to resource operation shape and field behavior.
- A resource should be creatable by POST on a collection and deletable by DELETE one path segment deeper.
- Avoid service behavior that rewrites user input or reorders fields/lists unless documented and unavoidable.
- Avoid returning partial or rewritten data that causes infrastructure-as-code tools to think drift exists.
- Passwords/secrets should usually be represented as references to secret resources.
