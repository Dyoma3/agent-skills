# Workflows

Step-by-step guides for common development tasks.

## Contents

- Create New Page
- Create New Component
- Create Dialog Component
- Add API Call
- Add Pinia Store
- Add Composable
- Add Zod Schema and Type
- Add Translations
- Refactor a Large Page

## Create New Page

1. **Create page file** `pages/my-page.vue`:

   ```vue
   <script setup lang="ts">
   const { data, loading } = useAxios({
     url: '/my-endpoint',
     schema: mySchema.array(),
   })
   </script>

   <template>
     <div class="px-8">
       <h2>{{ $t('myPage.title') }}</h2>
       <v-data-table :items="data ?? []" :loading="loading" />
     </div>
   </template>
   ```

2. **Add translations** to every locale file, when the repo has i18n.

3. **Add a route guard** in `middleware/` if the page is not public.

Keep the page at orchestration level. If a block of the page owns its own requests, watchers, and
derived state, it belongs in a feature component under `components/<Domain>/`.

---

## Create New Component

1. **Create component** under the domain that owns it, `components/<Domain>/MyComponent.vue`:

   ```vue
   <script setup lang="ts">
   import type { MyType } from '~/types'

   const props = defineProps<{
     item: MyType
   }>()

   const emit = defineEmits<{
     update: [item: MyType]
   }>()
   </script>

   <template>
     <v-card>
       <v-card-title>{{ item.name }}</v-card-title>
       <v-card-actions>
         <v-btn @click="emit('update', item)">
           {{ $t('common.save') }}
         </v-btn>
       </v-card-actions>
     </v-card>
   </template>
   ```

2. **Use in parent**:

   ```vue
   <domain-my-component :item="item" @update="handleUpdate" />
   ```

Use this workflow only when the extracted component is a real UI boundary. If the feature has its
own requests, watchers, derived state, or workflow-specific refs, move that logic into the component
too when it only serves that UI.

---

## Create Dialog Component

1. **Create dialog** `components/<Domain>/MyDialog.vue`:

   ```vue
   <script setup lang="ts">
   const isOpen = defineModel<boolean>()
   const emit = defineEmits<{ success: [] }>()

   const form = reactive({
     name: '',
     amount: 0,
   })

   async function handleSubmit() {
     await useAxios({
       url: '/endpoint',
       method: 'POST',
       data: form,
       lazy: true,
     }).fetch()

     emit('success')
     isOpen.value = false
   }
   </script>

   <template>
     <v-dialog v-model="isOpen" max-width="500">
       <v-card>
         <v-card-title>{{ $t('myDialog.title') }}</v-card-title>
         <v-card-text>
           <v-text-field v-model="form.name" :label="$t('common.name')" />
         </v-card-text>
         <v-card-actions>
           <v-btn @click="isOpen = false">{{ $t('common.cancel') }}</v-btn>
           <v-btn color="primary" @click="handleSubmit">
             {{ $t('common.save') }}
           </v-btn>
         </v-card-actions>
       </v-card>
     </v-dialog>
   </template>
   ```

2. **Use in parent**:

   ```vue
   <script setup>
   const isDialogOpen = ref(false)
   </script>

   <template>
     <v-btn @click="isDialogOpen = true">{{ $t('common.create') }}</v-btn>
     <domain-my-dialog v-model="isDialogOpen" @success="refresh" />
   </template>
   ```

The dialog owns the submit workflow and its own form state. The parent only opens it and reacts to
the domain event.

---

## Add API Call

```typescript
// GET request, fired on setup
const { data, loading, error, fetch } = useAxios({
  url: '/resources',
  schema: resourceSchema.array(),
})

// POST request, fired manually
const { fetch: createResource } = useAxios({
  url: '/resources',
  method: 'POST',
  data: resourceData,
  schema: resourceSchema,
  lazy: true,
})

await createResource()
```

Always pass a `schema`. An unvalidated response silently propagates backend contract drift into the
UI.

---

## Add Pinia Store

1. **Create store** `stores/my_store.ts`:

   ```typescript
   export const useMyStore = defineStore(
     'myStore',
     () => {
       const items = ref<Item[]>([])
       const loading = ref(false)

       async function fetchItems() {
         loading.value = true
         try {
           const { fetch } = useAxios({
             url: '/items',
             schema: itemSchema.array(),
             lazy: true,
           })
           items.value = (await fetch()) ?? []
         } finally {
           loading.value = false
         }
       }

       return { items, loading, fetchItems }
     },
     { persist: true }
   )
   ```

2. **Use in component**:

   ```typescript
   const store = useMyStore()
   await store.fetchItems()
   ```

Only reach for a store when the state is persisted or shared across distant parts of the app.

---

## Add Composable

1. **Create composable** `composables/useMyFeature.ts`:

   ```typescript
   export function useMyFeature(initialValue: string) {
     const value = ref(initialValue)
     const { t } = useI18n()

     const options = computed(() => [
       { title: t('option.one'), value: '1' },
       { title: t('option.two'), value: '2' },
     ])

     function reset() {
       value.value = initialValue
     }

     return { value, options, reset }
   }
   ```

2. **Use in component**:

   ```typescript
   const { value, options, reset } = useMyFeature('default')
   ```

Only create a composable when the extracted logic is one cohesive concern. If it is private to one
component or workflow, co-locate it in that component's folder instead of `composables/`.

---

## Add Zod Schema and Type

1. **Create schema** in `schemas/`:

   ```typescript
   import { z } from 'zod'

   export const mySchema = z.object({
     id: z.number(),
     name: z.string(),
     amount: z.number(),
     createdAt: z.string(),
   })
   ```

2. **Export type** in `types/index.ts`:

   ```typescript
   import type { mySchema } from '~/schemas'
   export type MyType = z.infer<typeof mySchema>
   ```

3. **Use with the API composable**:

   ```typescript
   const { data } = useAxios({
     url: '/endpoint',
     schema: mySchema.array(),
   })
   ```

The schema must match the backend response shape. When the backend payload changes, update the
schema in the same change.

---

## Add Translations

1. **Add the key to every locale file** the repo ships:

   ```json
   // i18n/locales/<locale>.json
   { "myKey": "My text" }
   ```

2. **Use in template**:

   ```vue
   {{ $t('myKey') }}
   ```

A key present in only some locales is a runtime fallback, not a translation.

---

## Refactor a Large Page

Use this workflow when a page starts containing several independent features or the script becomes
hard to follow.

1. **Identify the concerns** in the page.
   Example:
   - upload and parse file
   - create record
   - subscribe to update events
   - fetch list
   - select candidates

2. **Group the current page script by concern** before extracting anything.
   Each concern should contain its own:
   - refs
   - computed values
   - requests
   - actions
   - watchers

3. **Choose the right destination for each concern**:
   - component: when the concern owns a cohesive UI block and the logic exists only for that block
   - composable: when the concern is non-visual reactive logic with one responsibility
   - store: when the concern needs persistence or sharing across distant parts of the app
   - page: only for route-level coordination and layout

4. **Extract complete concerns, not partial fragments**.
   When moving a feature to a component, move the related template and the related logic together.

5. **Reduce the parent page to orchestration**.
   After extraction, the page should mostly wire feature blocks together and react to domain events.

6. **Check the resulting boundaries**.
   Signs the refactor is wrong:
   - the child needs many props and callback functions
   - the page still owns most of the child's requests and watchers
   - one composable now mixes unrelated concerns
   - persistence is handled with direct browser storage instead of Pinia

Example of a good split:

- `components/<Domain>/Uploader.vue`: upload UI, parsing state, record creation workflow
- `composables/<domain>/useUpdateSubscription.ts`: subscription lifecycle
- page: route-level layout, high-level orchestration, composition of feature blocks
