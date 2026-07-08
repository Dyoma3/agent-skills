---
name: adonis-v6-backend
description: "Use for all work in an AdonisJS v6 TypeScript backend: implementation, debugging, design, code review, architecture guidance, testing, routes, controllers, Lucid models, Zod validators, services, migrations, middleware, authorization, queues, MCP tools exposed through Adonis HTTP, and backend tests."
---

# Adonis v6 Backend

Use this skill as the generic AdonisJS v6 backend context.

## Stack

- AdonisJS v6
- TypeScript
- PostgreSQL
- Lucid ORM
- Redis
- BullMQ
- Zod

## Commands

- `npm run dev`: start the backend development server, commonly on port `3333`
- `npm run test`: run the test suite
- `npm run format`: apply formatting after edits
- `node ace migration:run`: run database migrations
- `node ace make:migration <name>`: create a new database migration

## Core Rules

- Keep controllers thin: validate input, resolve request context, authorize, and delegate larger
  workflows to models or services.
- Use Zod validators in `app/validators`. Do not introduce Vine.
- Use Adonis path aliases such as `#models`, `#validators`, `#lib`, `#start`, and `#config`.
- Use route lazy imports in `start/routes.ts`.
- Put application workflows in named services only when they represent a real domain/model
  composition or a large HTTP action. Do not use `app/services` as a generic helper bucket.
- Keep authorization in middleware and Bouncer policies instead of ad hoc controller checks.
- Do not add broad controller `try/catch` blocks around normal Lucid or service operations only to
  convert errors into generic HTTP responses. Let the global exception layer handle unexpected
  failures.
- When an Adonis HTTP workflow is reused by MCP, jobs, commands, or another transport, split the
  service into an HTTP adapter such as `httpExecute()` and a transport-neutral `execute(input)`
  method that returns plain data.
- Keep MCP tool definitions under `app/mcp`, not under `app/services`, and expose existing Adonis
  services instead of duplicating controller logic.
- Add or update tests when changing validation, business behavior, response shapes, authorization,
  MCP contracts, or persistence behavior.
- Run the repo's formatter after edits.

## References

Read only the relevant reference files for the task:

- `references/patterns.md` for naming, imports, controllers, Lucid models, validators, services,
  MCP tools, queues, middleware, routes, and data conventions.
- `references/workflows.md` for step-by-step implementation flows: endpoints, MCP tools,
  authorization, migrations, models, queues, relationships, middleware, and factories.
- `references/testing.md` when adding, changing, or reviewing tests.
- `references/anti-patterns.md` before reviews, refactors, architecture changes, or broad
  implementation work.
