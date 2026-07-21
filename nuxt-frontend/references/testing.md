# Testing

## Contents

- Setup
- Unit Testing Components
- Testing Composables
- Testing Stores
- E2E Testing
- Best Practices

## Setup

Nuxt frontends in this stack often have no formal test suite. Check the repo before assuming one
exists, and follow the repo's own testing context when it does.

When introducing tests, use:

- **Vitest** — unit testing (Vite-native)
- **Vue Test Utils** — component testing
- **Playwright** — E2E testing

## Unit Testing Components

```typescript
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import MyComponent from '~/components/MyComponent.vue'

describe('MyComponent', () => {
  it('renders correctly', () => {
    const wrapper = mount(MyComponent, {
      props: {
        item: { id: 1, name: 'Test' },
      },
    })

    expect(wrapper.text()).toContain('Test')
  })

  it('emits update event', async () => {
    const wrapper = mount(MyComponent, {
      props: { item: { id: 1, name: 'Test' } },
    })

    await wrapper.find('button').trigger('click')

    expect(wrapper.emitted('update')).toBeTruthy()
  })
})
```

Assert the component's public contract: rendered output and emitted domain events. A component that
owns a workflow should be tested through that workflow, not through its internal refs.

## Testing Composables

```typescript
import { describe, it, expect } from 'vitest'
import { useResourceSearch } from '~/composables/useResourceSearch'

describe('useResourceSearch', () => {
  it('initializes with default values', () => {
    const { searchType, search } = useResourceSearch({})

    expect(searchType.value).toBe('description')
    expect(search.value).toBe('')
  })
})
```

A composable with one concern is straightforward to test in isolation. If a composable is hard to
set up because it needs unrelated context, that is usually a sign it owns more than one concern.

## Testing Stores

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useSessionStore } from '~/stores/session'

describe('Session Store', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('logs in user', () => {
    const store = useSessionStore()

    store.login('token123')

    expect(store.isLogged).toBe(true)
    expect(store.token).toBe('token123')
  })

  it('logs out user', () => {
    const store = useSessionStore()
    store.login('token123')

    store.logout()

    expect(store.isLogged).toBe(false)
    expect(store.token).toBe('')
  })
})
```

Call `setActivePinia(createPinia())` in `beforeEach` so stores do not leak state between tests.

## E2E Testing

```typescript
import { test, expect } from '@playwright/test'

test('user can login', async ({ page }) => {
  await page.goto('/login')

  await page.fill('[data-testid="email"]', 'test@test.com')
  await page.fill('[data-testid="password"]', 'password')
  await page.click('[data-testid="submit"]')

  await expect(page).toHaveURL('/dashboard')
})
```

E2E tests are the right level for auth flows, route guards, and cross-page workflows.

## Best Practices

1. Add `data-testid` attributes for E2E selectors instead of relying on Vuetify's internal DOM.
2. Mock API calls in unit tests; do not hit a real backend.
3. Test user interactions and emitted events, not implementation details.
4. Keep tests focused on single behaviors.
5. When schema validation is part of the behavior, assert against a payload that would fail the
   schema, not only the happy path.
