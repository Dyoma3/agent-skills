# Workflows

Step-by-step guides for common Adonis backend development tasks.

## Contents

- Create New API Endpoint
- Add MCP Tool Reusing An Adonis Service
- Add Authorization For A Resource Endpoint
- Add Database Migration
- Add New Model
- Dispatch Background Job
- Add New Queue
- Add Model Relationship
- Add Custom Middleware
- Add Factory for Testing

## Create New API Endpoint

1. **Add route** in `start/routes.ts`:

   ```typescript
   router
     .group(() => {
       router.get('', [MyController, 'index'])
       router.post('', [MyController, 'store'])
     })
     .prefix('my-resource')
     .use([middleware.auth()])
   ```

2. **Create controller** in `app/controllers/`:

   ```typescript
   export default class MyController {
     async index({ auth }: HttpContext) {
       return await MyResource.query().where('userId', auth.user!.id)
     }

     async store({ request, auth, bouncer }: HttpContext) {
       const data = validateRequest(storeValidator, request.body())
       await bouncer.with(MyResourcePolicy).authorize('store')
       return await MyResource.create({ ...data, userId: auth.user!.id })
     }
   }
   ```

   Keep controllers on the happy path by default.
   Do not add `try/catch` around standard CRUD operations just to translate thrown errors into HTTP
   responses. Let Adonis handle unexpected exceptions unless the route explicitly requires custom
   handling for a known exception type.

   If a single action grows large because it coordinates validation, response shaping, several
   models, queues, or endpoint-specific branching, extract that workflow to an HTTP action service
   and delegate with Adonis method injection:

   ```typescript
   export default class MyController {
     @inject()
     store(_ctx: HttpContext, service: MyResourceStoreService) {
       return service.execute()
     }
   }
   ```

   The service may inject `HttpContext` when it owns request-specific orchestration. Return the
   service promise from the controller; the controller method only needs `async` when it has its own
   awaits or control flow.

   If the same action will be reused by another transport, split the service into `httpExecute()`
   and `execute(input)`. The controller should call `httpExecute()`. The transport-neutral
   `execute(input)` should return plain data and must not call `ctx.response`.

   Prefer existing ability names before introducing new ones:

   - `show` for single-resource reads
   - `update` for edits
   - domain-specific names only when needed by the action

   Example:

   ```typescript
   async show({ response, params, bouncer }: HttpContext) {
     const resource = await MyResource.findOrFail(params.id)
     await bouncer.with(MyResourcePolicy).authorize('show', resource)

     return response.status(200).json(resource)
   }
   ```

3. **Add validator** in `app/validators/`:

   ```typescript
   export const storeValidator = z.object({
     name: z.string().min(1),
     quantity: z.coerce.number().int().min(0),
   })
   ```

4. **Add middleware** as needed, such as auth, role checks, or repo-specific scoping middleware.

## Add MCP Tool Reusing An Adonis Service

1. **Create or update the service** with a transport-neutral execution method:

   ```typescript
   @inject()
   export default class ResourceIndexService {
     constructor(private ctx: HttpContext) {}

     async httpExecute() {
       return this.ctx.response.ok(await this.execute(this.ctx.request.qs()))
     }

     async execute(input: unknown) {
       const payload = validateRequest(resourceIndexValidator, input)
       const user = this.ctx.auth.user!
       const resources = await user
         .related('resources')
         .query()
         .paginate(payload.page, payload.pageSize)
       const serialized = resources.serialize()

       return {
         data: serialized.data,
         total: serialized.meta.total,
       }
     }
   }
   ```

   `execute(input)` is the shared path. It may validate, authorize, use middleware-provided request
   context, and query models. It should return plain data. For model-backed records, prefer Lucid
   `serialize()` output over hand-written DTOs. Do not add `dto.ts` or serializer helper files under
   `app/services` just to satisfy an MCP output schema.

2. **Delegate the HTTP controller** through Adonis method injection:

   ```typescript
   export default class ResourcesController {
     @inject()
     index(_ctx: HttpContext, service: ResourceIndexService) {
       return service.httpExecute()
     }
   }
   ```

3. **Create the MCP tool** under `app/mcp/tools`:

   ```typescript
   export function getResourcesTool(ctx: HttpContext): McpTool {
     return { name: 'get_resources', config, callback: createCallback(ctx) }
   }
   ```

   Keep `config` and `createCallback(ctx)` as top-level definitions in the tool file.

4. **Call the service from the tool callback** through the request container:

   ```typescript
   async function getResources(ctx: HttpContext, input: unknown) {
     const service = await ctx.containerResolver.make(ResourceIndexService)

     return service.execute(input)
   }
   ```

5. **Register the tool** in the MCP handler's tool list and let the existing registration loop call
   `server.registerTool(...)`.

6. **Update tests** for the HTTP endpoint and the MCP tool behavior. Prefer testing the tool result
   shape over testing the exact global tool list.

## Add Authorization For A Resource Endpoint

1. **Load the scoped resource**:

   ```typescript
   const resource = await Resource.findOrFail(params.id)
   ```

2. **Authorize with Bouncer** using an existing policy and ability name when possible:

   ```typescript
   await bouncer.with(ResourcePolicy).authorize('show', resource)
   ```

3. **Only add a new ability** when no existing ability name fits the action.

4. **Do not copy legacy manual scoping checks** into new endpoints unless the route truly does not
   fit the policy model.

## Add Database Migration

1. **Create migration**:

   ```bash
   node ace make:migration create_my_resources_table
   ```

2. **Define schema** in `database/migrations/`:

   ```typescript
   export default class extends BaseSchema {
     protected tableName = 'my_resources'

     async up() {
       this.schema.createTable(this.tableName, (table) => {
         table.increments('id')
        table.string('name').notNullable()
        table.timestamp('created_at')
        table.timestamp('updated_at')
       })
     }

     async down() {
       this.schema.dropTable(this.tableName)
     }
   }
   ```

3. **Run migration**:

   ```bash
   node ace migration:run
   ```

## Add New Model

1. **Create model** in `app/models/`:

   ```typescript
   export default class MyResource extends BaseModel {
     @column({ isPrimary: true })
     declare id: number

     @column()
     declare userId: number

     @column()
     declare name: string

     @column()
     declare quantity: number

     @column.dateTime({ autoCreate: true })
     declare createdAt: DateTime

     @column.dateTime({ autoCreate: true, autoUpdate: true })
     declare updatedAt: DateTime

     @belongsTo(() => User)
     declare user: BelongsTo<typeof User>
   }
   ```

2. **Add relationship** in parent model:

   ```typescript
   @hasMany(() => MyResource)
   declare myResources: HasMany<typeof MyResource>
   ```

## Dispatch Background Job

1. **Import queue**:

   ```typescript
   import syncQueue from '#lib/queues/sync'
   ```

2. **Add job** with options:

   ```typescript
   await syncQueue.add(
     'job-name',
     {
       resourceId: resource.id,
       date: '2025-01',
     },
     {
       attempts: 3,
       backoff: { type: 'exponential', delay: 1000 },
       removeOnComplete: 50,
       removeOnFail: 100,
     }
   )
   ```

## Add New Queue

1. **Create queue** in `lib/queues/`:

   ```typescript
   import { Queue } from 'bullmq'
   import redisConfig from '#config/redis'
   import app from '@adonisjs/core/services/app'

   const myQueue = new Queue('my-queue', {
     connection: redisConfig.connections.main,
   })

   app.terminating(() => myQueue.close())

   export default myQueue
   ```

2. **Add worker** in the worker application when the repo has one:

   ```typescript
   const myWorker = new Worker(
     'my-queue',
     async (job) => {
       const { default: processJob } = await import('./jobs/my_job/index.js')
       return processJob(job.data)
     },
     { connection: redisConfig }
   )
   ```

## Add Model Relationship

### HasMany

```typescript
// In parent
@hasMany(() => Resource)
declare resources: HasMany<typeof Resource>

// In child
@belongsTo(() => User)
declare user: BelongsTo<typeof User>
```

### ManyToMany

```typescript
// In both models
@manyToMany(() => Tag, { pivotTable: 'resource_tags' })
declare tags: ManyToMany<typeof Tag>
```

## Add Custom Middleware

1. **Create middleware** in `app/middleware/`:

   ```typescript
   export default class MyMiddleware {
     async handle({ auth, response }: HttpContext, next: NextFn) {
       // Check condition
       if (!condition) {
         return response.forbidden({ message: 'Access denied' })
       }
       return next()
     }
   }
   ```

2. **Register** in `start/kernel.ts`:

   ```typescript
   export const middleware = router.named({
     // ... existing
     myMiddleware: () => import('#middleware/my_middleware'),
   })
   ```

3. **Use in routes**:

   ```typescript
   router.get('/protected', [Controller, 'method']).use([middleware.myMiddleware()])
   ```

## Add Factory for Testing

Create in `database/factories/`:

```typescript
import Factory from '@adonisjs/lucid/factories'
import MyResource from '#models/my_resource'

export default Factory.define(MyResource, ({ faker }) => ({
  name: faker.commerce.productName(),
  quantity: faker.number.int({ min: 1, max: 100 }),
})).build()
```

Use in tests:

```typescript
const resource = await MyResourceFactory.merge({ userId: user.id }).create()
```
