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
node ace test --files "tests/functional/payments.spec.ts"  # Single file
node ace test --groups "Group name" # Group test
node ace test --tests "Test name" # Single test
```

## Test Structure

```bash
tests/
├── functional/     # API integration tests
│   ├── payments.spec.ts
│   ├── invoices.spec.ts
│   └── ...
├── mcp/            # MCP protocol and tool integration tests
│   ├── tools/
│   ├── projects/
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
├── projects/
└── companies/
```

`tests/mcp/tools` is for generic MCP tool registry requests such as `tools/list`. Put individual
tool behavior under the domain folder that owns the tool, for example
`tests/mcp/projects/get_projects.spec.ts`.

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
import { CompanyFactory } from '#database/factories/company_factory'
import { PaymentTypes } from '#types/index'
import User from '#models/user'
import Company from '#models/company'

let user: User
let company: Company

test.group('Payments create', async (group) => {
  group.setup(async () => {
    await ace.exec('db:mytruncate', [])
    user = await UserFactory.create()
    company = await CompanyFactory.create()
    await company.addUser(user.id)
  })

  const payload = {
    projectId: null,
    amount: 1000,
    date: '2021-01-01',
    type: PaymentTypes.Income,
    hasIva: false,
    isFictional: false,
    balanceAtPayment: 5000,
  }

  test('creates a payment', async ({ assert, client }) => {
    const response = await client.post('/payments').json(payload).loginAs(user)

    response.assertStatus(201)
    assert.isNumber(response.body().id)
    assert.containsSubset(response.body(), { ...payload, companyId: company.id })
  })

  test('requires company membership', async ({ client }) => {
    const newUser = await UserFactory.create()
    const response = await client.post('/payments').json(payload).loginAs(newUser)

    response.assertStatus(403)
  })

  test('rejects invalid payloads', async ({ client }) => {
    const response = await client
      .post('/payments')
      .json({ ...payload, amount: 'invalid' })
      .loginAs(user)

    response.assertStatus(422)
  })
})
```

## Testing HTTP Action Services

HTTP action services that inject `HttpContext` are usually best covered through functional tests for
the endpoint, because validation, middleware, auth/company context, and response serialization are
part of the behavior.

When testing an action service directly, create or bind the request-specific dependencies through
the Adonis container the same way the HTTP stack does. Do not manually construct a service with
mocked partial context objects unless the test is intentionally isolated and the required context
surface is very small.

Example of per-test reset when each test mutates shared state heavily:

```typescript
test.group('Project.getReadyItems', (group) => {
  group.each.setup(async () => {
    await ace.exec('db:mytruncate', [])
  })

  test('returns unique ready items belonging to the project', async ({ assert }) => {
    // create data specific to this test
  })
})
```

## Using Factories

Factories are in `database/factories/`:

```typescript
// Create single record
const payment = await PaymentFactory.create()

// Create with specific values
const payment = await PaymentFactory.merge({ companyId: company.id, amount: 50000 }).create()

// Create multiple
const payments = await PaymentFactory.merge({ companyId: company.id }).createMany(5)

// With relationships
const user = await UserFactory.with('company')
  .with('payments', 3)
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
response.assertBodyContains({ amount: 10000 })

// Array length
response.assertBodyContains({ length: 3 })

// Specific structure
response.assertBody({
  id: 1,
  amount: 10000,
  date: '2025-01-15',
})
```

## Testing with Authentication

```typescript
// Login as user
const response = await client.get('/payments').loginAs(user)

// Admin routes
const admin = await UserFactory.create()
const response = await client.get('/admin/users').loginAs(admin)

// Worker auth
const response = await client
  .post('/payments/worker-batch')
  .header('Authorization', `Bearer ${process.env.WORKER_SECRET}`)
  .json(data)
```

## Database Assertions

```typescript
import Payment from '#models/payment'

test('creates payment in database', async ({ client, assert }) => {
  const user = await UserFactory.create()
  const company = await CompanyFactory.create()
  await company.addUser(user.id)

  await client.post('/payments').loginAs(user).json({
    projectId: null,
    amount: 10000,
    date: '2025-01-15',
    type: PaymentTypes.Income,
    hasIva: false,
    isFictional: false,
    balanceAtPayment: 5000,
  })

  const payment = await Payment.query().where('companyId', company.id).first()

  assert.isNotNull(payment)
  assert.equal(payment!.amount, 10000)
})
```

## Factory Definitions

Example factory in `database/factories/`:

```typescript
import Factory from '@adonisjs/lucid/factories'
import Payment from '#models/payment'
import { CompanyFactory } from '#database/factories/company_factory'
import { ItemFactory } from '#database/factories/item_factory'
import { PaymentTypes } from '#types/index'

export default Factory.define(Payment, ({ faker }) => ({
  amount: faker.number.int({ min: 1000, max: 1000000 }),
  date: faker.date.recent().toISOString().split('T')[0],
  type: faker.helpers.enumValue(PaymentTypes),
  hasIva: faker.datatype.boolean(),
  isFictional: false,
  description: faker.commerce.productDescription(),
}))
  .relation('company', () => CompanyFactory)
  .relation('items', () => ItemFactory)
  .build()
```
