# Coding Patterns

## Contents

- Source Directory
- Naming Conventions
- Imports
- Components
- Component Directory Organization
- Vuetify Components
- API Calls
- Pinia Stores
- Zod Schemas and Types
- i18n
- Composables
- Separation of Concerns in Files
- Choosing Between Page, Component, Composable, and Store
- Functions, Classes, and Free Helpers
- Dialogs
- CSS

## Source Directory

Nuxt 3 defaults to project-root directories (`components/`, `pages/`, `composables/`). Nuxt 4
defaults to an `app/` source directory (`app/components/`, `app/pages/`). Both layouts use the same
`~/` alias and the same conventions. Follow whatever the repo already does; this file writes paths
without the prefix.

## Naming Conventions

| Element           | Style                             | Example                        |
| ----------------- | --------------------------------- | ------------------------------ |
| Pages             | kebab-case                        | `admin-resources.vue`          |
| Components        | PascalCase path segments          | `components/<Domain>/Table.vue` |
| Component domains | PascalCase product or feature owner | `Billing`                      |
| Composables       | camelCase + `use`                 | `useAxios.ts`                  |
| Stores            | camelCase                         | `session.ts`                   |

## Imports

Nuxt auto-imports (no import needed):

```typescript
ref, computed, watch, useRoute, useRouter, navigateTo, defineStore
```

Path aliases:

```typescript
import type { Resource } from '~/types'
import { resourceSchema } from '~/schemas'
import { ResourceTypes } from '~/enums'
```

## Components

Always use `<script setup lang="ts">`:

```vue
<script setup lang="ts">
import type { Resource } from '~/types'

const props = defineProps<{
  resource: Resource
  isEditing?: boolean
}>()

const emit = defineEmits<{
  update: [resource: Resource]
  delete: [id: number]
}>()
</script>

<template>
  <v-card>
    <v-card-title>{{ resource.description }}</v-card-title>
    <v-card-actions>
      <v-btn @click="emit('update', resource)">{{ $t('common.save') }}</v-btn>
    </v-card-actions>
  </v-card>
</template>
```

For complex features, components must represent a real feature boundary, not just extracted markup.

- Move the feature's reactive state, requests, watchers, and derived values together with the
  template when that logic only serves the extracted UI.
- Keep the parent page responsible only for orchestration that truly spans multiple concerns.
- Prefer smaller, cohesive component APIs over passing many props and callbacks just to keep logic
  in the parent.
- If a component owns a workflow, it should usually emit domain events such as `created`, `saved`,
  `reset`, not low-level UI implementation details.

Template refs:

```typescript
const tableRef = useTemplateRef<typeof ResourcesTable>('table-ref')
```

## Component Directory Organization

Use domain-first organization for `components/`. A domain is the product area, business concept, or
bounded feature that owns the component. The primary folder should describe that owner, not the UI
shape.

Preferred structure, using placeholder names:

```txt
components/
  Common/
    ConfirmationDialog.vue
    DatePicker.vue
    PeriodPicker.vue
    TabNavigator.vue
  <Domain>/
    Table.vue
    CreateDialog.vue
    UpdateDialog.vue
    UploadDialog.vue
    DownloadButton.vue
  <AnotherDomain>/
    List.vue
    DetailDialog.vue
    <Feature>/
      Panel.vue
      Chart.vue
    <WorkflowDialog>/
      index.vue
      FirstTab.vue
      SecondTab.vue
```

Directory rules:

- Put app-specific components under the domain that owns the concept.
- Use `Common` only for reusable UI primitives that are not tied to one business concept.
- Put route-specific feature components under `components/<Domain>/<Feature>/`, not under `pages/`.
- Use nested folders when a workflow is complex enough to have private child components, such as
  `components/<Domain>/<Workflow>/`.
- Keep the root of `components/` limited to truly app-level components.
- Avoid primary folders based only on UI shape or action, such as `Dialogs`, `Tables`, `Charts`,
  `Upload`, `Download`, or `Sync`.

Nuxt auto-import names should make the file location easy to infer. These names are illustrative,
not prescribed domains:

- `components/Billing/Table.vue` -> `<billing-table />`
- `components/Billing/UploadDialog.vue` -> `<billing-upload-dialog />`
- `components/Catalog/Import/Preview.vue` -> `<catalog-import-preview />`
- `components/Orders/CreateDialog/index.vue` -> `<orders-create-dialog />`

Auto-import is a convenience, not a requirement. If a deeply nested component produces a noisy
template name, import it explicitly and use a short local alias in that file.

Component-private support code should live next to the component when it is not reusable outside
that component or workflow.

```txt
components/
  <Domain>/
    <Workflow>/
      index.vue
      useWorkflowForm.ts
      helpers.ts
      types.ts
      ChildPanel.vue
```

Co-location rules:

- Use `components/<Domain>/<Workflow>/` when a component has private child components, helper
  functions, local types, or one-off composables.
- Import co-located `.ts` files explicitly with relative imports. They are private support modules,
  not Nuxt auto-imported global composables.
- Move code to `composables/` only when it is reusable or independently meaningful outside the
  component folder.
- Move code to `lib/` only when it is framework-independent and clearly reusable.
- Keep shared backend response shapes, schemas, and widely used types in the shared `schemas/` and
  `types/` areas.

## Vuetify Components

Common components:

- `v-btn`, `v-icon` (mdi-\* icons)
- `v-card`, `v-dialog`, `v-menu`
- `v-text-field`, `v-select`, `v-checkbox`
- `v-data-table`

Layout utilities: `d-flex`, `ga-3`, `px-8`, `mb-16`

## API Calls

Backend calls go through the repo's HTTP composable, which wraps Axios, attaches the session token,
and validates the response with a Zod schema. Do not call Axios directly from a component when the
composable exists.

```typescript
const { data, loading, error, fetch } = useAxios({
  url: '/resources',
  method: 'GET',
  schema: resourceSchema.array(),
  lazy: true, // To avoid auto-fetch
})

await fetch() // Fetch manually
```

The composable returns `data`, `loading`, `error`, and `fetch`. Requests are fired on setup unless
`lazy: true` is passed, so use `lazy` for anything triggered by a user action.

## Pinia Stores

Setup store syntax:

```typescript
export const useSessionStore = defineStore(
  'session',
  () => {
    const isLogged = ref(false)
    const token = ref('')

    function login(newToken: string) {
      token.value = newToken
      isLogged.value = true
    }

    return { isLogged, token, login }
  },
  { persist: true }
)
```

Use Pinia for persisted client state.

- Do not access `localStorage` or `sessionStorage` directly in pages, components, or composables.
- If state must survive reloads or be shared across feature boundaries, create or extend a store.
- Keep stores focused on one domain or feature area. Do not turn a store into a generic dump of
  unrelated state.

## Zod Schemas and Types

Schemas mirror backend response shapes and are the single source of truth for types.

```typescript
// schemas/resource.ts
export const resourceSchema = z.object({
  id: z.number(),
  amount: z.number(),
  date: z.string(),
  type: z.nativeEnum(ResourceTypes),
})

// types/index.ts
export type Resource = z.infer<typeof resourceSchema>
```

Check the repo's Zod major version before using version-specific APIs. Zod 3 and Zod 4 differ on
enum, error, and coercion helpers.

## i18n

When the repo has i18n, all user-facing text must use it:

```vue
<template>
  <v-btn>{{ $t('common.save') }}</v-btn>
</template>

<script setup>
const { t } = useI18n()
const label = computed(() => t('resources.amount'))
</script>
```

Use `$t` in templates and `t` from `useI18n()` in script when the string feeds computed values or
component options. A new key must be added to every locale file the repo ships.

## Composables

```typescript
export function useResourceSearch(params: Params) {
  const searchType = ref<'amount' | 'description'>('description')
  const search = ref('')

  const { t } = useI18n()
  const options = computed(() => [{ title: t('common.amount'), value: 'amount' }])

  return { searchType, search, options }
}
```

Composables must own a single logic concern.

- Good examples: subscription lifecycle, feature polling, form submission, table filtering,
  pagination, feature-specific selection state.
- Bad examples: mixing upload parsing, polling, selection persistence, and dialog UI state in one
  composable.
- A composable may coordinate refs, computed state, watchers, requests, and side effects, but all of
  them must belong to the same concern.
- If two parts of a composable could be extracted independently without becoming awkward, they are
  probably different concerns.

Before creating a composable, check:

- Is this one cohesive concern, not a bundle of unrelated refs/functions?
- Is it reusable or independently meaningful outside the current component?
- Does it hide a complete concept rather than simply moving code out of a file?
- Does it avoid owning state that naturally belongs to a specific child component?

## Separation of Concerns in Files

Within a page or component, group code by logic concern instead of by primitive type.

Preferred grouping:

1. app setup and injections
2. feature A state, computed values, requests, actions, watchers
3. feature B state, computed values, requests, actions, watchers
4. lifecycle glue that coordinates those concerns

For non-trivial pages and components, add short section comments that describe the logic concern
handled by the code that follows.

Recommended format:

```typescript
// ### APP SETUP ###
// ### FEATURE A ###
// ### FEATURE B ###
// ### LIFECYCLE GLUE ###
```

Use the concern itself in the comment text, not primitive buckets such as `refs`, `computed`,
`watchers`, or `methods`.

Avoid this structure for large files:

1. all refs
2. all computed values
3. all requests
4. all watchers
5. all methods

The preferred structure makes it easier to:

- read a feature end to end
- extract a concern into a composable
- extract a concern into a component
- see what state and requests belong together

## Choosing Between Page, Component, Composable, and Store

Use a page for route-level orchestration and layout.

- Pages should compose feature blocks and coordinate route-level concerns.
- Pages should not retain all feature logic once a cohesive feature has been extracted.
- Pages and parent components should not become controllers for child UI internals. They should own
  route params, selected records that are genuinely shared across children, coarse layout, and
  explicit data passed between feature components.

Use a component for a cohesive UI plus the logic that only exists to support that UI.

- If the extracted code still needs many props and callbacks to work, the boundary is probably
  wrong.
- A list component should usually own its own loading, search, filtering, hover state, and selection
  state when those only serve that list.
- A preview or visualization component should usually own its own derived visual data, projection,
  clustering, and rendering rules when those only serve that preview.
- Prefer domain events from the owning child, such as `active-item-change`, over lifting
  child-specific state into the parent.

Use a composable for non-visual reactive logic with one concern.

- Good fits: subscriptions, feature polling, search/filter state, form orchestration, selection
  behavior.
- Composables can be used by a page or by a component.

Use a store for persisted state or state shared across distant parts of the app.

- Prefer a store when reload persistence is needed.
- Prefer a store when multiple pages or unrelated components depend on the same feature state.
- Do not create a store for temporary local state that belongs to a single component or page
  feature.

## Functions, Classes, and Free Helpers

These rules apply inside any TS file (page, component, composable, store, `lib/`).

Split a long function into named helpers when it runs through multiple distinct phases or holds more
state than a reader can comfortably track. The helper names should describe the phases — do not
replace structure with comments.

Group tightly coupled functions around an explicit shared context when they share state or
setup-time inputs. Choose the smallest fitting shape: a class, a factory/closure, a focused
composable for reactive Vue concerns, or a local helper object.

- Strong signals: an external handle threaded through every call, indexes built once and read many
  times, durable shared context used by several helpers, or the same 3+ arguments repeating across
  helper signatures.
- For classes, the constructor receives the shared inputs and (when useful) builds the indexes;
  instance methods consume them.
- Prefer simple immutable return values or one local accumulator in an orchestration function when
  that keeps the flow clear.
- Use static methods on a class only for stateless transforms that conceptually belong to the same
  class concern.

Keep stateless, single-step transformations as free functions. Do not wrap them in a class for
namespacing alone.

## Dialogs

```vue
<script setup>
const isOpen = defineModel<boolean>()
const emit = defineEmits<{ success: [] }>()

async function handleSubmit() {
  await createResource()
  emit('success')
  isOpen.value = false
}
</script>

<template>
  <v-dialog v-model="isOpen" max-width="600">
    <v-card><!-- content --></v-card>
  </v-dialog>
</template>
```

Dialogs own their open state through `defineModel`, and report outcomes to the parent with a domain
event. The parent should not drive the dialog's internal workflow.

## CSS

- Scoped styles in components
- Global styles in the app's main stylesheet
- Prefer Vuetify utility classes
