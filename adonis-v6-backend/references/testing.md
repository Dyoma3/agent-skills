# Testing

## Contents

- Commands
- Test Structure
- Writing MCP Tests
- Writing Functional Tests
- Testing HTTP Action Services
- Using Factories
- Test Assertions
- Testing with Authentication
- Database Assertions
- Factory Definitions

## Commands

```bash
npm run test                    # Run all tests
node ace test functional        # Functional tests only
node ace test mcp               # MCP protocol/tool tests only
node ace test unit              # Unit tests only
node ace test --files "tests/functional/resources.spec.ts"  # Single file
node ace test --groups "Group name" # Group test
node ace test --tests "Test name" # Single test
```

## Test Structure

```bash
tests/
├── functional/     # API integration tests
│   ├── resources.spec.ts
│   ├── users.spec.ts
│   └── ...
├── mcp/            # MCP protocol and tool integration tests
│   ├── tools/
│   ├── resources/
│   └── ...
├── unit/           # Unit tests
│   └── ...
└── bootstrap.ts    # Test setup
```

## Writing MCP Tests

Use the `mcp` suite for MCP protocol requests and tool calls. Keep the folder structure
domain-oriented, similar to the functional suite:

```bash
tests/mcp/
├── auth/
├── tools/
├── resources/
└── users/
```

`tests/mcp/tools` is for generic MCP tool registry requests such as `tools/list`. Put individual
tool behavior under the domain folder that owns the tool, for example
`tests/mcp/resources/get_resources.spec.ts`.

MCP tests still use the HTTP test client when the MCP server is exposed through `/mcp`, but they
should assert MCP protocol results: JSON-RPC envelopes, SSE data events, `structuredContent`, and
tool `content`.

## Writing Functional Tests

Use Japa with the repo's database reset pattern. If the repo has a truncate command, reset database
state with that command inside setup hooks.

Most files use `group.setup` to reset once and create shared fixtures for the whole group. Use
`group.each.setup` only when tests in the same file need a fresh database before every test.

```typescript
import { test } from '@japa/runner'
import ace from '@adonisjs/core/services/ace'
import { UserFactory } from '#database/factories/user_factory'
import User from '#models/user'
import Resource from '#models/resource'
import { ResourceFactory } from '#database/factories/resource_factory'

let user: User

test.group('Resources create', async (group) => {
  group.setup(async () => {
    await ace.exec('db:truncate', [])
    user = await UserFactory.create()
  })

  const payload = {
    name: 'Example resource',
    quantity: 10,
    date: '2021-01-01',
  }

  test('creates a resource', async ({ assert, client }) => {
    const response = await client.post('/resources').json(payload).loginAs(user)

    response.assertStatus(201)
    assert.isNumber(response.body().id)
    assert.containsSubset(response.body(), { ...payload, userId: user.id })
  })

  test('requires authentication', async ({ client }) => {
    const response = await client.post('/resources').json(payload)

    response.assertStatus(401)
  })

  test('rejects invalid payloads', async ({ client }) => {
    const response = await client
      .post('/resources')
      .json({ ...payload, quantity: 'invalid' })
      .loginAs(user)

    response.assertStatus(422)
  })
})
```

## Testing HTTP Action Services

HTTP action services that inject `HttpContext` are usually best covered through functional tests for
the endpoint, because validation, middleware, auth/request context, and response serialization are
part of the behavior.

When testing an action service directly, create or bind the request-specific dependencies through
the Adonis container the same way the HTTP stack does. Do not manually construct a service with
mocked partial context objects unless the test is intentionally isolated and the required context
surface is very small.

Example of per-test reset when each test mutates shared state heavily:

```typescript
test.group('Resource.getReadyRecords', (group) => {
  group.each.setup(async () => {
    await ace.exec('db:truncate', [])
  })

  test('returns unique ready records belonging to the resource', async ({ assert }) => {
    // create data specific to this test
  })
})
```

## Using Factories

Factories are in `database/factories/`:

```typescript
// Create single record
const resource = await ResourceFactory.create()

// Create with specific values
const resource = await ResourceFactory.merge({ userId: user.id, quantity: 50 }).create()

// Create multiple
const resources = await ResourceFactory.merge({ userId: user.id }).createMany(5)

// With relationships
const user = await UserFactory
  .with('resources', 3)
  .create()
```

## Test Assertions

```typescript
// Status codes
response.assertStatus(200)
response.assertStatus(401)
response.assertStatus(404)

// Body content
response.assertBodyContains({ id: 1 })
response.assertBodyContains({ quantity: 10 })

// Array length
response.assertBodyContains({ length: 3 })

// Specific structure
response.assertBody({
  id: 1,
  quantity: 10,
  date: '2025-01-15',
})
```

## Testing with Authentication

```typescript
// Login as user
const response = await client.get('/resources').loginAs(user)

// Admin routes
const admin = await UserFactory.create()
const response = await client.get('/admin/users').loginAs(admin)

// Token-protected internal route
const response = await client
  .post('/resources/internal-batch')
  .header('Authorization', `Bearer ${process.env.INTERNAL_API_TOKEN}`)
  .json(data)
```

## Database Assertions

```typescript
import Resource from '#models/resource'

test('creates resource in database', async ({ client, assert }) => {
  const user = await UserFactory.create()

  await client.post('/resources').loginAs(user).json({
    name: 'Example resource',
    quantity: 10,
    date: '2025-01-15',
  })

  const resource = await Resource.query().where('userId', user.id).first()

  assert.isNotNull(resource)
  assert.equal(resource!.quantity, 10)
})
```

## Factory Definitions

Example factory in `database/factories/`:

```typescript
import Factory from '@adonisjs/lucid/factories'
import Resource from '#models/resource'
import { UserFactory } from '#database/factories/user_factory'
import { TagFactory } from '#database/factories/tag_factory'

export default Factory.define(Resource, ({ faker }) => ({
  name: faker.commerce.productName(),
  quantity: faker.number.int({ min: 1, max: 100 }),
  date: faker.date.recent().toISOString().split('T')[0],
  description: faker.commerce.productDescription(),
}))
  .relation('user', () => UserFactory)
  .relation('tags', () => TagFactory)
  .build()
```
