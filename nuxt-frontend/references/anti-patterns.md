# Anti-Patterns

These are common mistakes that make a Nuxt frontend harder to read, refactor, and maintain.

## State Persistence

- Do not use raw `localStorage` or `sessionStorage` directly in pages, components, or composables.
- Do not create ad hoc storage keys inside feature files when the persisted state should belong to a
  store.
- Use Pinia persistence for client-side persisted state.

## Component Extraction

- Do not extract only the HTML into a child component while leaving all related logic in the parent.
- Do not create components whose only purpose is to reduce file length.
- Do not pass large sets of props and callback functions when the child should own the feature
  workflow.
- Do not emit low-level implementation events if a higher-level domain event would make the boundary
  clearer.

## Component Organization

- Do not create primary component folders by UI shape or action, such as `Dialogs`, `Tables`,
  `Charts`, `Upload`, `Download`, or `Sync`.
- Do not put page-specific Vue components under `pages/`; use `components/<Domain>/<Feature>/`.
- Do not use `Common` as a dumping ground for business components whose owner is unclear.
- Do not move component-private helpers, types, or one-off composables into global `composables/` or
  `lib/` folders just to shorten a component.

## Composables

- Do not mix multiple logic concerns in one composable.
- Do not move unrelated watchers, requests, and refs into a composable just because the file is
  large.
- Do not create a composable if the logic is tightly coupled to a single extracted component and is
  not reused or independently meaningful.
- Do not create a broad composable that bundles multiple logical concerns just to reduce file size
  or move code out of a component.
- Do not create a composable whose name implies one concern while its return value exposes state and
  actions for unrelated workflows, such as fetching data, managing search, owning hover/selection,
  computing visualization markers, and clustering.

## Parent Components

- Do not lift child-specific state into the parent just because two child components need to
  communicate.
- Do not make a parent own a child component's search input, filtering, hover state, rendering
  rules, or loading state when the child owns the workflow and renders the state.
- Do not pass many low-level props and callbacks between parent and child when a child can emit a
  single domain event and own the rest of its workflow.

## Pages

- Do not organize large page scripts as separate global sections for all refs, all computed values,
  all requests, and all methods.
- Do not keep feature-specific API calls in the page after extracting that feature into a component
  or composable.
- Do not let route pages become the storage location for every concern in a feature.

## Stores

- Do not use a store for temporary local UI state that belongs to one component.
- Do not put unrelated feature state into the same store.
- Do not bypass the store with direct browser storage APIs.

## API Calls and Schemas

- Do not call Axios directly from a component when the repo has an HTTP composable that attaches
  auth and validates responses.
- Do not skip the Zod schema on a request; an unvalidated response hides backend contract drift.
- Do not hand-write a TypeScript interface that duplicates an existing schema instead of using
  `z.infer`.
- Do not let a schema drift from the backend payload it mirrors.

## i18n

- Do not hardcode user-facing strings in templates or scripts when the repo has i18n.
- Do not add a key to one locale file and leave the others missing it.

## Functions and Classes

- Do not leave a multi-phase function as one block when the phases have distinct names. Extract
  named helpers; do not just add comments.
- Do not extract free helpers that all take the same 3+ arguments by hand. Consider a class,
  factory, focused composable, or local helper object that owns the shared inputs.
- Do not wrap a single stateless transform in a class only for namespacing.
- Do not split a function only to reduce its line count when no real phase boundary exists.

## Refactors

- Do not refactor by moving markup first and deciding logic ownership later.
- Do not keep old boundaries once a better feature boundary has been introduced.
- Do not split code mechanically by file type; split it by responsibility.
