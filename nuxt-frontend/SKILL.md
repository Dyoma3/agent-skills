---
name: nuxt-frontend
description: "Use for all work in a Nuxt 3 or Nuxt 4 TypeScript frontend: implementation, debugging, design, code review, architecture guidance, Vue components, pages, layouts, composables, Pinia stores, Zod schemas, API calls, Vuetify UI, i18n, refactors, and frontend tests."
---

# Nuxt Frontend

Use this skill as the generic Nuxt frontend context. It applies to both Nuxt 3 and Nuxt 4; the
repo's own context owns version-specific details such as source directory layout and Zod major
version.

## Stack

- Nuxt 3 or Nuxt 4
- Vue 3 with the Composition API
- TypeScript
- Vuetify 3
- Pinia
- Zod
- Axios

## Commands

- `npm run dev`: start the frontend development server, commonly on port `3000`
- `npm run build` / `npm run generate`: production build
- `npm run typecheck`: type checking

## Core Rules

- Always use `<script setup lang="ts">` with typed `defineProps`, `defineEmits`, and `defineModel`.
- Organize `components/` domain-first. The primary folder names the business concept that owns the
  component, never the UI shape (`Dialogs`, `Tables`, `Charts`).
- Extract components at real feature boundaries. Move the feature's state, requests, watchers, and
  derived values together with its template when that logic only serves the extracted UI.
- A component that owns a workflow emits domain events (`created`, `saved`, `reset`), not low-level
  UI implementation details.
- Give every composable exactly one logic concern. Do not bundle unrelated refs, requests, and
  watchers into a composable to shorten a file.
- Use Pinia for persisted or widely shared client state. Never touch `localStorage` or
  `sessionStorage` directly from a page, component, or composable.
- Validate every API response with a Zod schema, and derive TypeScript types from those schemas with
  `z.infer`. Do not hand-write a type that duplicates a schema.
- Group code inside a file by logic concern, not by primitive type. Do not lay out large scripts as
  one block of refs, then all computed values, then all requests, then all methods.
- Keep pages at route-level orchestration and layout. A page should not remain the controller for a
  child component's internals.
- In any TS file, split multi-phase functions into named helpers, and group tightly coupled helpers
  around an explicit shared context instead of threading the same 3+ arguments through each call.
- When the repo has i18n, all user-facing text goes through it, and a new key is added to every
  locale file in the same change.
- Run the repo's formatter and type check after edits.

## References

Read only the relevant reference files for the task:

- `references/patterns.md` for naming, component organization, components, composables, stores,
  schemas, API calls, Vuetify, i18n, file structure, and the page/component/composable/store
  decision.
- `references/workflows.md` for step-by-step implementation flows: pages, components, dialogs, API
  calls, stores, composables, schemas, translations, and large-page refactors.
- `references/testing.md` when adding, changing, or reviewing tests.
- `references/anti-patterns.md` before reviews, refactors, architecture changes, or broad
  implementation work.
