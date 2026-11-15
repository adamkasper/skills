---
name: nuxt-fsd
description: >
  Feature-Sliced Design (FSD) architecture for Nuxt 4+ projects. Use when
  deciding where to place new code, which layer a module belongs to, how to
  structure slices and segments, handle cross-slice communication, or when
  scaffolding new Nuxt pages/features/entities. Covers layer mapping to Nuxt
  conventions, the thin-page routing pattern, `src/` as FSD root, auto-import
  strategy, composable patterns, and server-side considerations. Activates on
  mentions of FSD, feature-sliced, layers, slices, architecture decisions, or
  when creating new modules in a Nuxt project that follows FSD structure.
---

# Feature-Sliced Design for Nuxt 4+

Official FSD docs: <https://feature-sliced.design/>
LLM reference: <https://feature-sliced.design/llms-full.txt>

## Core principle

FSD organizes code into **layers** with a strict dependency rule: **each layer can only import from layers below it, never above or sideways**. This prevents tangled dependencies — a widget never knows about a page that uses it, a feature never reaches into another feature, and an entity never depends on the UI that displays it.

**FSD lives exclusively in `src/`.** The Nuxt `app/` directory is the runtime shell — it consumes FSD code from `src/` but is not itself organized by FSD. Things like `app/composables/`, `app/middleware/`, `app/plugins/`, and `app/layouts/` follow Nuxt conventions, not FSD layers.

## FSD layers (all in `src/`)

Layers are ordered top (most specific) to bottom (most stable). Every layer can **only import from layers below it**.

```
pages      ← FSD page slices in src/pages/ — compose widgets, features, entities
widgets    ← Self-contained UI blocks with coupled logic + presentation
features   ← Reusable user interactions — logic is standalone, UI is replaceable
entities   ← Business domain models: types, schemas, base queries, formatters
shared     ← Framework utilities, UI kit, API client, helpers — zero business logic
```

### Layer quick-reference

| Layer | Contains | Does NOT contain |
|-------|----------|------------------|
| **shared** | UI kit components, `cn()`, date/string helpers, API client setup, type utilities | Business logic, domain concepts |
| **entities** | `User`, `Product`, `Order` types, Zod schemas, base `useFetch` wrappers, model formatters | User actions, interactive features |
| **features** | Auth flow, search logic, cart operations, form submission composables | Entity definitions, layout concerns |
| **widgets** | `ProductCard`, `AppHeader`, `Sidebar`, complete composed sections with own data | Raw entity data access, route logic |
| **pages** | Page slices assembling widgets/features, page-specific data fetching | Reusable pieces (extract when proven) |

---

## Nuxt 4+ directory mapping

```
project-root/
├── app/          ← Nuxt runtime shell, NOT FSD (thin pages, layouts, plugins, middleware, composables)
├── src/          ← FSD root: pages/, widgets/, features/, entities/, shared/
├── server/       ← Nuxt server routes, outside FSD
└── nuxt.config.ts
```

The full tree with example slices, the TypeScript includes `src/` needs, optional path aliases, the segments of the `shared` layer and the migration plan for an existing project are in [references/project-setup.md](references/project-setup.md). Read it when you set up or reconfigure a project.

### Key mapping rules

1. **`src/`** is the **FSD root**. All FSD layers (`pages/`, `widgets/`, `features/`, `entities/`, `shared/`) live here.
2. **`app/`** is the **Nuxt runtime shell**. It follows Nuxt conventions, not FSD. It consumes FSD code from `src/` via imports.
3. **`app/pages/`** contains **thin routing shells** only — they delegate to `src/pages/` page slices. See "Thin page pattern" below.
4. **`server/`** is outside FSD entirely. Server routes follow Nuxt server conventions.

---

## Thin page pattern

`app/pages/*.vue` files are **routing shells** (5–25 lines). They exist solely for Nuxt's file-based routing and delegate all real implementation to FSD page slices in `src/pages/`.

A thin page:
- Imports the page component from `src/pages/<slice>`
- Defines `definePageMeta({ i18n: { ... } })` for i18n path mappings (if using `@nuxtjs/i18n` with `customRoutes: 'meta'`)
- Defines `definePageMeta()` for validation, layout, route key
- Renders the imported page component

### Example thin page

```vue
<!-- app/pages/products/[id].vue — thin routing shell -->
<script setup lang="ts">
import { ProductDetailPage } from '~~/src/pages/product-detail'

definePageMeta({
  layout: 'default',
  validate: async (route) => /^\d+$/.test(route.params.id as string),
  i18n: {
    paths: {
      cs: '/produkty/[id]',
      en: '/products/[id]',
    },
  },
})
</script>

<template>
  <ProductDetailPage />
</template>
```

The actual implementation lives in the FSD page slice:

```
src/pages/product-detail/
  ui/
    ProductDetailPage.vue    ← Full page implementation
  model/
    useProductDetail.ts
  index.ts                   ← Exports ProductDetailPage
```

---

## FSD page slices in `src/pages/`

`src/pages/` contains full FSD page slices — **not** Nuxt file-based routes. These are regular FSD slices with `ui/`, `model/`, `api/`, and `index.ts`.

### Conventions

- **Slice naming**: kebab-case matching the domain concept: `product-detail/`, `blog-article-detail/`, `change-email/`
- **Slice groups**: see the next section. Parentheses are only a folder for related page slices, e.g. `(user-settings)/change-email`.
- **Shared chrome / page layouts**: Widgets (`src/widgets/user-settings-shell`, same idea as `wizard-shell`). Simple chrome without domain logic can live in `shared/ui` or Nuxt `app/layouts`. Never `_layout/` and never `ui/` / `model/` / `index.ts` at the group root.

### Page slice structure

```
src/pages/product-detail/
  ui/
    ProductDetailPage.vue      ← Main page component
    ProductGallery.vue
    ProductSpecs.vue
  model/
    useProductDetail.ts        ← Page-specific composable
  api/
    queries.ts                 ← Page-specific data fetching
  index.ts                     ← Public API: exports ProductDetailPage
```

```ts
// src/pages/product-detail/index.ts
export { default as ProductDetailPage } from './ui/ProductDetailPage.vue'
```

---

## Slice groups

Official FSD ([slice groups](https://feature-sliced.design/docs/reference/slice-groups), [slices and segments](https://feature-sliced.design/docs/reference/slices-segments#slice-groups)): a group is **only a navigation folder**. It is **not a slice**.

> A slice group is not a slice. It does not have its own segments like `model`, `ui`, or `api`, nor does it have a public API such as `index.ts`. Shared code used across multiple slices should not be placed in a slice group either.

> Closely related slices can be structurally grouped in a folder, but they should exercise the same isolation rules as other slices — there should be **no code sharing** in that folder.

In this repo, group folders use parentheses so they are obvious: `(user-settings)`, `(company-settings)`, `(offer-detail)`, `(offer-extend)`, `(blog)`.

```
src/pages/(user-settings)/     ← group: ONLY page slices
  change-email/                ← slice (ui/, model/, index.ts)
  two-fa/

src/widgets/user-settings-shell/   ← shared chrome (sidebar, hub, page wrapper)
  ui/
  model/
  index.ts
```

**Wrong** (group pretending to be a slice):

```
src/pages/(user-settings)/
  index.ts          ← no
  ui/               ← no
  model/            ← no
  _layout/          ← no
  change-email/
```

A folder **without** parentheses is itself a page slice and must have FSD segments under it.

---

## Imports and aliases

### Primary pattern: `~~/src/` prefix

Use Nuxt's `~~/` alias (which resolves to the project root) to import from FSD layers:

```ts
// CORRECT — primary import pattern
import { useAuth } from '~~/src/features/auth'
import { type User } from '~~/src/entities/user'
import { UiButton } from '~~/src/shared/ui'
import { ProductDetailPage } from '~~/src/pages/product-detail'
```

You can also configure shorter aliases such as `@shared` in `nuxt.config.ts`, see [references/project-setup.md](references/project-setup.md). Both patterns are valid.

**Important**: Do NOT rely on Nuxt auto-imports for cross-layer dependencies. Always use explicit imports through the slice's public API (`index.ts`). This keeps dependencies visible and enforceable.

Nuxt auto-imports are acceptable **only** for:
- Vue/Nuxt built-ins (`ref`, `computed`, `useFetch`, `useRoute`, `navigateTo`, etc.)
- Global app-layer composables in `app/composables/`

---

## Slices and segments

A **slice** is a subfolder inside a layer, named after a business domain concept:
- `src/features/auth`, `src/entities/user`, `src/widgets/product-card`

A **segment** groups code by technical role inside a slice:

```
src/features/auth/
  ui/          ← Vue components
  model/       ← Composables, stores (Pinia), state, types
  api/         ← Data fetching (useFetch, $fetch, query wrappers)
  lib/         ← Helpers specific to this slice
  config/      ← Constants, feature flags
  index.ts     ← Public API (REQUIRED)
```

### Naming conventions

- **kebab-case** everywhere: `user-profile`, `add-to-cart`, `product-card`
- Domain-first names, NOT technical: `auth` not `use-auth-hook`, `product` not `product-helpers`
- Vue components: **PascalCase** filenames matching component name: `LoginForm.vue`, `ProductCard.vue`
- Composables: **camelCase** with `use` prefix: `useAuth.ts`, `useProductSearch.ts`

---

## Public API

Every slice **must** expose an `index.ts`. External code imports only from `index.ts`.

```ts
// src/features/auth/index.ts
export { default as LoginForm } from './ui/LoginForm.vue'
export { useAuth } from './model/useAuth'
export type { AuthUser, AuthCredentials } from './model/types'
```

**Deep imports are forbidden between slices:**
```ts
// WRONG
import { useAuth } from '~~/src/features/auth/model/useAuth'

// CORRECT
import { useAuth } from '~~/src/features/auth'
```

Within-slice relative imports are fine:
```ts
// Inside src/features/auth/ui/LoginForm.vue
import { useAuth } from '../model/useAuth'
```

---

## Pages-first workflow

Start everything in `src/pages/`. Extract only when reuse is **proven**.

```
New functionality needed
        │
        v
  Build inside src/pages/<slice>/
        │
        v
  Used in a 2nd place?
     NO --> Keep in src/pages/
     YES
        │
        v
  Duplicate it (2 copies is acceptable)
        │
        v
  Used in a 3rd place?
     NO --> Keep duplicated
     YES
        │
        v
  Extract. Ask: "Is this logic always tied to THIS specific UI?"
     YES --> Widget (logic + UI coupled)
      NO --> Feature (logic reusable, UI replaceable)
             or Entity (pure domain data, no user action)
```

**Rule**: Extract when you have evidence (3+ usages), not a prediction. Premature extraction creates unnecessary abstraction.

Splitting a page into small **local** `ui/` parts (header, form, list, footer) is not an extraction, so it doesn't wait for 3+ usages. Split when a part has its own responsibility (own props, own state, its own place in the variants). A small component that changes as one unit stays one file. Extraction to `widgets/` / `features/` still waits for 3+ usages. How to assemble those parts: `composition-over-configuration` skill.

---

## Widget vs Feature decision

The critical question: **"Can the logic be used WITHOUT this specific UI?"**

### Widget — logic and UI are inseparable

- Always rendered as a unit
- Contains its own data fetching and state
- Reused across 2+ pages but always as a whole block

```
src/widgets/job-list/
  ui/
    JobList.vue          ← Fetches data, renders list, handles pagination
    JobListItem.vue      ← Presentational sub-component
  model/
    useJobList.ts        ← Encapsulated composable, not exported alone
  index.ts               ← Exports only <JobList />
```

### Feature — logic is standalone

- Composable works with any UI
- Default UI is provided but replaceable
- Logic can be used headlessly

```
src/features/search/
  model/
    useSearch.ts         ← Works standalone, any UI can consume it
  ui/
    SearchBar.vue        ← Default UI, receives state via props/inject
  index.ts               ← Exports both useSearch and SearchBar
```

### Quick decision table

| Module | Widget or Feature? | Why |
|--------|-------------------|-----|
| Search with debounced results | Feature | `useSearch` powers different UIs |
| Infinite scroll product list | Widget | Fetch + render always together |
| Auth login form | Feature | `useAuth` reusable in modal, page, drawer |
| Navigation sidebar | Widget | Nav items + layout always coupled |
| Like/bookmark button | Feature | `useLike` works standalone |
| Dashboard stats panel | Widget | Chart + data always together |

---

## State management placement

All state lives in the `model/` segment of the relevant slice:

| Pattern | When to use | Placement |
|---------|------------|-----------|
| Simple composable (`useState`) | Shared state within a feature/entity, SSR-safe | `src/features/auth/model/useAuth.ts` |
| Provider/Inject (`createInjectionState`) | State scoped to a component subtree | `src/features/cart/model/useCart.ts` |
| Pinia store | Global state, devtools, persistence, complex actions | `src/entities/user/model/store.ts` |

### Provider/Inject pattern

Use `createInjectionState` from `@vueuse/core` for scoped state. Always export an `OrThrow` variant so consumers get a clear error if the provider is missing. Each provider instance owns its state, and consumers get readonly state plus actions:

```ts
// src/features/cart/model/useCart.ts
import { createInjectionState } from '@vueuse/core'
import { readonly, shallowRef } from 'vue'

const [useProvideCart, useCart] = createInjectionState(() => {
  const items = shallowRef<readonly CartItem[]>([])
  const add = (item: CartItem) => { items.value = [...items.value, item] }
  return { items: readonly(items), add }
})

export { useProvideCart }

export function useCartOrThrow() {
  const state = useCart()
  if (!state) throw new Error('useCartOrThrow must be used inside a cart provider')
  return state
}
```

## Data fetching placement

| What | Where | Example path |
|------|-------|-------------|
| Base queries (read) | Entity `api/` segment | `src/entities/product/api/queries.ts` |
| Mutations (write) | Feature `api/` segment | `src/features/add-to-cart/api/mutations.ts` |
| Page-specific fetching | Page slice `api/` or inline | `src/pages/product-detail/api/queries.ts` |

---

## Import rules (strict)

### Layer imports — only downward (within `src/`)

```
pages     can import from → widgets, features, entities, shared
widgets   can import from → features, entities, shared
features  can import from → entities, shared
entities  can import from → shared
shared    can import from → (nothing — only external packages)
```

`app/` (Nuxt shell) can import from any FSD layer in `src/`.

### Same-layer isolation

Slices within the same layer **cannot** import from each other:

```ts
// WRONG — features/auth importing from features/profile
import { useProfile } from '~~/src/features/profile'

// CORRECT — extract shared concept to entities or shared
import { type User } from '~~/src/entities/user'
```

### Cross-entity references (@x notation)

When entities have legitimate business relationships, use explicit `@x` cross-references:

```ts
// src/entities/order/ui/OrderCard.vue
import { UserAvatar } from '~~/src/entities/user/@x/order'
```

The `@x/<consumer>` folder is a controlled cross-import API. Use sparingly.

---

## Cross-slice communication patterns

When slices on the same layer need to coordinate:

1. **Extract to a lower layer** — if two features share a concept, move it to `entities` or `shared`
2. **Event-based communication** — use a shared event bus (`mitt`) or Pinia store for loose coupling
3. **Composition at a higher layer** — let a page or widget compose multiple features together

---

## Nuxt `app/` directory (not FSD)

Everything in `app/` follows **Nuxt conventions**, not FSD. These files consume FSD code from `src/` but are not themselves organized into FSD layers:

| Nuxt directory | Role | Relation to FSD |
|----------------|------|-----------------|
| `app/pages/` | Thin routing shells | Imports page components from `src/pages/` |
| `app/layouts/` | Nuxt layouts | May import widgets from `src/widgets/` |
| `app/middleware/` | Route middleware | May import composables from `src/features/` or `src/entities/` |
| `app/plugins/` | Global setup | May import from any `src/` layer |
| `app/composables/` | Global composables | Only truly app-wide concerns, not FSD-sliced code |
| `server/` | Server routes | Entirely outside FSD, follows Nuxt server conventions |

---

## Scaffolding a new slice

```
src/features/bookmark/
  ui/
    BookmarkButton.vue
  model/
    useBookmark.ts
    types.ts
  api/
    mutations.ts
  index.ts              ← Public API (REQUIRED)
```

Only create segments you actually need. An entity with only types needs only `model/` and `index.ts`.

---

## Common mistakes

Eleven mistakes that come up in review, each with the fix, are in [references/common-mistakes.md](references/common-mistakes.md). Read it when you review FSD code.

---

## Checklist for new code

Before writing or reviewing code, verify:

- [ ] Which layer does this belong to? (Use the decision tree)
- [ ] Does the slice have an `index.ts` public API?
- [ ] Are all cross-slice imports going through `index.ts`?
- [ ] Am I importing only from layers below?
- [ ] Are imports using `~~/src/` prefix (or configured aliases)?
- [ ] Is `app/pages/*.vue` a thin routing shell (<25 lines)?
- [ ] Widget or Feature? (Asked: "can logic exist without this UI?")
- [ ] Is extraction proven (3+ usages) or premature?
- [ ] Is all FSD-structured code in `src/`, not in `app/` or `server/`?
- [ ] Does `app/` only contain Nuxt runtime concerns (thin pages, layouts, plugins, middleware)?
- [ ] If this is a `(group)/`, does it contain only page slices (no group-root `ui/` / `model/` / `index.ts` / `_layout/`)?
- [ ] Does shared page chrome live in `src/widgets/*-shell`, imported via that widget's public API?
