# Coding Patterns

## Contents

- Naming Conventions
- Imports
- Controllers
- Models
- Validators
- Services
- Services Shared By HTTP And Other Transports
- MCP Tools
- Queues
- Middleware
- Routes
- Data Conventions

## Naming Conventions

| Element               | Style      | Example                             |
| --------------------- | ---------- | ----------------------------------- |
| Files                 | snake_case | `payment_states_controller.ts`      |
| Classes               | PascalCase | `PaymentStatesController`           |
| Database columns      | snake_case | `company_id`, `created_at`          |
| TypeScript properties | camelCase  | `companyId`, `createdAt`            |
| Routes                | kebab-case | `/payment-states`, `/bank-accounts` |

## Imports

Use AdonisJS path aliases:

```typescript
import Company from '#models/company'
import { storeValidator } from '#validators/payment'
import syncQueue from '#lib/queues/sync'
import env from '#start/env'
import redisConfig from '#config/redis'
```

## Controllers

Controllers are thin: validate input, delegate to services or models:

```typescript
export default class PaymentsController {
  async store({ request, auth, bouncer }: HttpContext) {
    const data = validateRequest(storeValidator, request.body())
    const company = auth.user!.currentCompany
    await bouncer.with(PaymentPolicy).authorize('store', company)
    return await company.related('payments').create(data)
  }
}
```

Keep controllers on the happy path.
Do not add broad `try/catch` blocks around normal `findOrFail`, `create`, `save`, `delete`, or
service calls just to translate thrown exceptions into `badRequest` or `internalServerError`
responses.
Let Adonis handle unexpected exceptions through the global exception layer.

Only catch in a controller when custom handling is explicitly required for a known exception type
or route contract. In that case, catch narrowly, map intentionally, and document why the custom
handling belongs in the controller.

Authorization belongs in middleware and Bouncer policies, not in ad hoc controller checks.
For user-authenticated resource endpoints, prefer loading the resource and authorizing it with
Bouncer instead of repeating manual ownership checks in the controller.

Policy ability names should match the repo's resource-action conventions:

- `show` for read access to a single resource
- `store` for create actions when a policy is needed
- `update` for edits
- `delete` or `destroy` only if the policy already uses that naming
- domain-specific names only when the action is not standard CRUD

Example:

```typescript
async show({ response, company, params, bouncer }: HttpContext) {
  const project = await Project.findOrFail(params.id)
  await bouncer.with(ProjectPolicy).authorize('show', project, company!)

  return response.status(200).json(project)
}
```

Routes use lazy imports:

```typescript
const PaymentsController = () => import('#controllers/payments_controller')
```

## Models

Lucid models use `declare` for columns:

```typescript
export default class Payment extends BaseModel {
  @column({ isPrimary: true })
  declare id: number

  @column()
  declare amount: number

  @column.dateTime({ autoCreate: true })
  declare createdAt: DateTime

  @belongsTo(() => Company)
  declare company: BelongsTo<typeof Company>

  @manyToMany(() => Item, { pivotTable: 'items_payments' })
  declare items: ManyToMany<typeof Item>

  @beforeSave()
  static async validate(payment: Payment) {
    // validation logic
  }

  @computed()
  get formattedAmount() {
    return this.amount.toLocaleString('es-CL')
  }
}
```

## Validators

Use Zod schemas, not Vine.js:

```typescript
import { z } from 'zod'
import { financialAmountSchema, taxIdSchema } from '#lib/schemas'

export const storeValidator = z.object({
  amount: financialAmountSchema.pipe(z.number().min(0)),
  date: z.string().min(1),
  type: z.nativeEnum(PaymentTypes),
  hasIva: z.boolean(),
  issuerTaxId: taxIdSchema.nullable().optional(),
})
```

Validator conventions:

- For `PATCH` validators, prefer optional fields or `baseSchema.partial()`. Do not carry store or
  create defaults into update schemas unless the endpoint explicitly requires them.
- Prefer field-level `.optional()`, `.nullable()`, and `.default(...)` over object-level
  `.transform()` when the goal is simple value normalization.
- Extract helper schemas when they improve readability, represent a meaningful domain concept, or
  are reused. Keep schemas inline when extraction only adds indirection.

Common schema patterns:

- amount schemas that accept string and number input
- id schemas that coerce string query params to integers
- domain identifier schemas such as tax IDs when the repo uses them

## Services

Services in `app/services` cover two Adonis application patterns:

- Domain/model composition services
- HTTP action services resolved with dependency injection

Both patterns should name a concrete application workflow. Do not use `app/services` as a generic
bucket for unrelated helpers or scripts.

### Domain/Model Composition Services

Domain/model composition services are for business behavior that extends model capabilities
through composition. They are typically:

- Called from model methods to keep models clean
- Used when logic is too complex for a single model method
- Shared across multiple model methods with related domain behavior
- Independent from `HttpContext`

```typescript
// app/services/project/clone.ts
export default class ProjectCloneService {
  constructor(private project: Project) {}

  async execute(name: string) {
    // Complex cloning logic with transactions
  }
}

// Called from model method:
// app/models/project.ts
async clone(name: string) {
  const projectCloneService = new ProjectCloneService(this)
  return await projectCloneService.execute(name)
}
```

Domain services can also have static methods for aggregation:

```typescript
// app/services/item_period_stats.ts
export default class ItemPeriodStatsService {
  constructor(
    private item: Item,
    private startDate: string,
    private endDate: string
  ) {}

  async getStats(options: AmountUnitOptions = {}) {
    /* ... */
  }

  static aggregateStats(stats: Stats[]) {
    // Aggregate multiple item stats
  }
}
```

### HTTP Action Services

HTTP action services are Adonis request workflows extracted from controller methods. Use them when
a controller action would otherwise become large because it needs request validation, response
serialization, multiple model calls, policies, queues, or request-specific branching that does not
belong on a model.

These services may inject `HttpContext` through the Adonis container. Keep the controller method as
a thin delegation layer and return the service call:

```typescript
// app/controllers/oauth_controller.ts
import { inject } from '@adonisjs/core'
import type { HttpContext } from '@adonisjs/core/http'
import OauthApproveAuthorizationService from '#services/oauth/approve_authorization'

export default class OauthController {
  @inject()
  approveAuthorization(_ctx: HttpContext, service: OauthApproveAuthorizationService) {
    return service.execute()
  }
}
```

```typescript
// app/services/oauth/approve_authorization.ts
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'

@inject()
export default class OauthApproveAuthorizationService {
  constructor(private ctx: HttpContext) {}

  async execute() {
    const payload = validateRequest(approveAuthorizationValidator, this.ctx.request.body())
    // Orchestrate request-specific workflow and return the HTTP response.
    return this.ctx.response.redirect(redirectUrl)
  }
}
```

The controller method does not need `async` when it only returns the service promise. Dependency
injection resolves the service; it does not await service methods automatically. Return the result
so Adonis can handle it like any other controller response.

Prefer HTTP action services when:

- The logic is specific to one HTTP endpoint
- Validation, authorization, serialization, and response shape are part of the workflow
- The action coordinates several models, services, queues, or external resources
- Moving the code to a model would mix HTTP concerns into domain behavior

Prefer a domain/model service instead when:

- The behavior is reusable outside HTTP
- A model method should expose the behavior as part of the domain API
- The service should not know about request, response, auth, params, or session state

Do not use `app/services` as a generic bucket for non-application orchestration.

Do not put these in `app/services`:

- Ace command helpers and CLI orchestration that are not part of application behavior
- Evaluation runners, fixture loaders, or benchmark harnesses
- Test-only setup code
- Generic scripts that are not called by models, controllers, or production application flows

Place those according to scope instead:

- feature-specific eval or fixture code: under `evals/` near the dataset
- general reusable utilities: under `lib/`
- test support code: under `tests/`
- command-specific orchestration: next to the command or in a command-support folder

Placement rule:

- If the code is not something a model, controller, or production application flow would call, it
  probably does not belong in `app/services`.

Examples:

- Good: `app/services/project/clone.ts` used by `Project.clone()`
- Good: `app/services/project/ifc_recommendation.ts` used by a project model workflow
- Good: `app/services/oauth/approve_authorization.ts` used by one large HTTP action through Adonis
  method injection
- Bad: an eval runner under `app/services/project/*` when it only exists to support an Ace command

### Services Shared By HTTP And Other Transports

When an Adonis request workflow must be reused by another transport, keep the service as the shared
application boundary and split HTTP response handling from transport-neutral execution.

Use this shape:

- `httpExecute()` is the HTTP adapter. It reads from `ctx.request`, calls `execute(input)`, and
  returns an Adonis `ctx.response.*` result.
- `execute(input)` is transport-neutral. It validates explicit input, authorizes with the
  request-scoped context, performs model work, and returns plain data.

```typescript
import { inject } from '@adonisjs/core'
import { HttpContext } from '@adonisjs/core/http'
import validateRequest from '#lib/request_validator'
import { resourceIndexValidator } from '#validators/resource'

@inject()
export default class ResourceIndexService {
  constructor(private ctx: HttpContext) {}

  async httpExecute() {
    const result = await this.execute(this.ctx.request.qs())

    return this.ctx.response.ok(result)
  }

  async execute(input: unknown) {
    const { page, pageSize } = validateRequest(resourceIndexValidator, input)
    const user = this.ctx.auth.user!

    const resources = await user.related('resources').query().paginate(page, pageSize)
    const serialized = resources.serialize()

    return {
      data: serialized.data,
      total: serialized.meta.total,
    }
  }
}
```

The HTTP controller remains a thin delegation layer:

```typescript
export default class ResourcesController {
  @inject()
  index(_ctx: HttpContext, service: ResourceIndexService) {
    return service.httpExecute()
  }
}
```

Other transports resolve the same service through the request container and call `execute(input)`.
For example, an MCP tool callback can do this:

```typescript
async function getResources(ctx: HttpContext, input: unknown) {
  const service = await ctx.containerResolver.make(ResourceIndexService)

  return service.execute(input)
}
```

Keep the transport-neutral `execute(input)` free of `ctx.response` calls. It may use validators,
Bouncer, authenticated users, middleware-provided request context, and Lucid `findOrFail` methods.
Let thrown exceptions propagate naturally; each transport decides how to render them.

If a route or transport entry point is protected by middleware that guarantees request context, use
a non-null assertion for that context inside the service instead of repeating the same runtime
check. First verify that every caller uses the required middleware.

Validators reused by HTTP and non-HTTP transports should accept both HTTP query-string values and
native JSON values when needed. For example, pagination schemas may accept both numeric strings and
numbers.

## MCP Tools

MCP tool definitions live under `app/mcp`, not under `app/services`. Services remain for Adonis
application workflows; MCP tools are transport adapters that expose those workflows to MCP.

Use one file per tool under `app/mcp/tools`. Each tool exports a factory that receives the current
`HttpContext` and returns a typed `McpTool` object:

```typescript
import type { HttpContext } from '@adonisjs/core/http'
import { z } from 'zod'
import ResourceIndexService from '#services/resource/index'
import { resourceIndexValidator } from '#validators/resource'
import type { McpTool } from './types.js'

// ### TOOL ###

export function getResourcesTool(ctx: HttpContext): McpTool {
  return { name: 'get_resources', config, callback: createCallback(ctx) }
}

// ### CONFIG ###

const resourceSchema = z.strictObject({
  id: z.number(),
  name: z.string(),
})

const outputSchema = {
  data: z.array(resourceSchema),
  total: z.number(),
}

const resourceOutput = z.strictObject(outputSchema)

const config = {
  title: 'Get resources',
  description: 'Returns resources visible to the authenticated user.',
  inputSchema: resourceIndexValidator.shape,
  outputSchema,
  annotations: {
    readOnlyHint: true,
    destructiveHint: false,
    idempotentHint: true,
    openWorldHint: false,
  },
} satisfies McpTool['config']

// ### CALLBACK ###

function createCallback(ctx: HttpContext): McpTool['callback'] {
  return async (input) => {
    const structuredContent = resourceOutput.parse(await getResources(ctx, input))

    return {
      content: [
        {
          type: 'text' as const,
          text: JSON.stringify(structuredContent, null, 2),
        },
      ],
      structuredContent,
    }
  }
}

async function getResources(ctx: HttpContext, input: unknown) {
  const service = await ctx.containerResolver.make(ResourceIndexService)

  return service.execute(input)
}
```

Keep the `name`, `config`, and `callback` readable as separate top-level pieces. Do not define the
callback inline inside the returned object.

Use `z.strictObject(...)` for MCP output schemas and parse the returned data before assigning
`structuredContent`. MCP outputs are contracts, not loose mirrors of Lucid models. Strict schemas
make drift visible: if a model starts serializing a new field and the tool returns raw serialized
models, the tool test fails until the MCP output contract is intentionally updated. If the tool
should expose only a stable subset, map the service result to a DTO first, then strict-parse that
DTO. Do not rely on default `z.object(...)` behavior for output parsing, because it strips
undeclared serialized fields and can hide contract drift.

Register tools from the MCP handler by keeping a list of tool objects and iterating over it:

```typescript
private registerTools(server: McpServer) {
  for (const tool of this.tools) {
    server.registerTool(tool.name, tool.config, tool.callback)
  }
}
```

The MCP handler owns transport setup and registration. Tool files own MCP schemas and callback
adaptation. Shared application logic belongs in Adonis services or models, not in the MCP handler.

## Queues

Define queues in `lib/queues/`:

```typescript
import { Queue } from 'bullmq'
import redisConfig from '#config/redis'
import app from '@adonisjs/core/services/app'

const syncQueue = new Queue('sync', {
  connection: redisConfig.connections.main,
})

app.terminating(() => syncQueue.close())

export default syncQueue
```

Dispatch with options:

```typescript
await syncQueue.add('sync', payload, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
  removeOnComplete: 50,
  removeOnFail: 100,
})
```

## Middleware

Custom middleware in `app/middleware/`:

```typescript
export default class HasCompanyMiddleware {
  async handle({ auth, response }: HttpContext, next: NextFn) {
    if (!auth.user?.currentCompany) {
      return response.unauthorized({ message: 'No company selected' })
    }
    return next()
  }
}
```

Register in `start/kernel.ts`:

```typescript
export const middleware = router.named({
  auth: () => import('#middleware/auth_middleware'),
  hasCompany: () => import('#middleware/has_company_middleware'),
})
```

## Routes

Group routes with middleware:

```typescript
router
  .group(() => {
    router.get('', [Controller, 'index'])
    router.post('', [Controller, 'store'])
    router.patch(':id', [Controller, 'update'])
    router.delete(':id', [Controller, 'destroy'])
  })
  .prefix('resource-name')
  .use([middleware.auth(), middleware.hasCompany()])
```

Worker routes use worker-auth middleware when the repo has worker callbacks.

When adding a new user-authenticated route:

- confirm the route has the correct middleware
- confirm authorization is handled by a Bouncer policy when the route acts on a scoped resource
- confirm the policy ability name matches the existing repo convention before adding a new one

## Data Conventions

### Financial Amounts

- Store as integers in the database.
- Use the repo's shared amount schema for input validation.
- If the repo has a money column decorator, use it for price or amount model columns so consume and
  serialize behavior stays consistent.

```typescript
import { moneyColumn } from '#lib/decorators/money_column'

@moneyColumn()
declare price: number

@moneyColumn({ nullable: true })
declare optionalPrice: number | null
```

### Dates

- Use Luxon `DateTime`.
- Store timestamps as PostgreSQL `timestamp` columns unless the repo has a more specific rule.
- Preserve existing external date formats for integration payloads.
