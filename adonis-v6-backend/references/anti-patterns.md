# Anti-Patterns

These are common backend mistakes that make Adonis applications harder to reason about, test, and
evolve safely.

## Contents

- Controllers
- Authorization Boundaries
- Models and Services
- MCP Tools
- Validation and Schemas
- Routes and Middleware
- Queues and Worker Contracts
- Database and Migrations
- Refactors

## Controllers

- Do not put business logic, multi-step workflows, or heavy data transformations in controllers.
- Do not access many unrelated models directly from a controller when the logic belongs in a service
  or model method.
- Do not let controllers become the place where resource-scoping decisions are manually repeated.
- Do not keep a large endpoint-specific workflow in a controller when an Adonis HTTP action service
  would make the action clearer.
- Do not wrap normal controller persistence operations in broad `try/catch` blocks.
- Do not convert unknown exceptions into `response.badRequest(error.message)` or generic
  controller-level error responses.
- Do not handle framework, ORM, or database errors in controllers unless the endpoint explicitly
  requires a custom exception mapping.

## Authorization Boundaries

- Do not skip middleware or policy checks just because the route is internal or admin-facing.
- Do not mix worker-authenticated behavior with user-authenticated behavior without making the
  boundary explicit.
- Do not implement authorization ad hoc in controllers when middleware or Bouncer policies should
  own it.
- Do not copy existing manual ownership checks into new endpoints when a Bouncer policy should own
  the authorization decision.
- Do not introduce new policy ability names such as `view` when the repo already uses a
  conventional name such as `show` for the same kind of action.

## Models and Services

- Do not move complex orchestration into models when it spans several entities or side effects.
- Do not create bloated service classes that mix unrelated concerns.
- Do not duplicate the same domain logic across controllers, models, and services.
- Do not hide important business rules in scattered utility functions when they belong to a named
  domain service.
- Do not inject `HttpContext` into reusable domain/model services. Keep `HttpContext` limited to
  HTTP action services.
- Do not hide reusable domain rules inside HTTP action services just because the first caller is a
  controller.
- Do not call `ctx.response` from a service method intended to be reused by MCP, jobs, commands, or
  any non-HTTP transport. Keep response handling in a dedicated HTTP wrapper such as
  `httpExecute()`.
- Do not make controllers extract request input and pass it into an HTTP action service that
  already owns the request workflow. Let the service's HTTP wrapper read `ctx.request`.
- Do not create `dto.ts`, `serializer.ts`, `serialize_*`, or `to*Dto` files under `app/services`
  that merely copy fields from a Lucid model. Models already own their public serialization through
  `model.serialize()`, collection serialization, computed properties, and column serializers.
- Do not manually rebuild a model response field-by-field unless the endpoint intentionally owns a
  different contract. If a service needs to add workflow-specific fields or omit an internal
  preloaded relation, keep that small shaping local to the workflow and start from
  `model.serialize()`.
- Do not put MCP-only output DTOs in `app/services`. MCP contracts belong in `app/mcp/tools`
  schemas; services should return domain data or serialized model data, not transport-only helper
  shapes.

## MCP Tools

- Do not put tool implementations directly in the MCP handler. The handler should set up transport
  and register tools.
- Do not put MCP tool files under `app/services`; tools are transport adapters, not application
  services.
- Do not duplicate controller or service queries inside MCP callbacks when an Adonis service can
  expose a transport-neutral `execute(input)` method.
- Do not define large inline tool objects with nested callback logic when separate `name`, `config`,
  and callback definitions would keep the tool readable.
- Do not use `ctx.response` inside an MCP tool callback or inside the transport-neutral service
  method it calls.
- Do not add broad `try/catch` blocks to MCP callbacks just to translate validator, Bouncer, or
  Lucid errors. The MCP SDK represents normal tool execution failures as tool errors; add custom
  mapping only when the tool contract requires it.
- Do not use loose MCP output schemas that silently strip undeclared fields. Use strict output
  schemas so Lucid serialization drift fails tests instead of changing or hiding the tool contract
  accidentally.
- Do not use `.passthrough()` for MCP output schemas unless the tool contract explicitly allows
  arbitrary extension fields.
- Do not satisfy a strict MCP schema by creating a service-layer DTO helper that mirrors a model.
  Update the MCP schema to match the intentional serialized model contract, or perform a small
  transport-local projection when the tool deliberately exposes a subset.

## Validation and Schemas

- Do not introduce Vine.js validators.
- Do not accept loosely typed payloads when a Zod schema should define the contract.
- Do not duplicate shared coercion logic for numbers, ids, or domain identifiers instead of reusing
  shared schemas.
- Do not rely on implicit casting when request validation should make the transformation explicit.
- Do not use object-level `.transform()` only to convert missing fields to `null` or apply simple
  field defaults.
- Do not encode authorization, source restrictions, or resource mutability rules in validators.
  Handle those in controllers, policies, or domain/model logic.
- Do not introduce schema helpers that hide simple field definitions without improving readability.

## Routes and Middleware

- Do not register routes without the middleware needed for the route's authentication,
  authorization, and resource-scoping model.
- Do not expose worker-only endpoints through normal user-authenticated routes.

## Queues and Worker Contracts

- Do not change queue payloads without checking the corresponding worker consumer.
- Do not change worker callback or result payloads without checking the backend endpoint that
  receives them.
- Do not enqueue jobs from controllers when the payload or retry behavior has not been thought
  through.
- Do not scatter queue naming or queue configuration outside `lib/queues`.

## Database and Migrations

- Do not add tables or relationships without checking how they fit the existing domain model.
- Do not create migrations that omit important foreign-key behavior without a clear reason.
- Do not encode business rules only in migrations when they also need application-level validation.

## Refactors

- Do not refactor by moving code mechanically between files without clarifying ownership of the
  concern.
- Do not split code only to reduce file length if the new boundaries are not domain-meaningful.
- Do not leave partially moved logic behind after introducing a service or model method.
- Do not change response shapes, route contracts, or serialized fields silently during refactors.
