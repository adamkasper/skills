# Project setup and migration

Read this when you set up FSD in a Nuxt project, change its config, or move existing code into `src/`.

## Contents

- Full directory layout
- TypeScript configuration
- Optional path aliases
- Shared layer structure
- Migration strategy

## Full directory layout

```
project-root/
├── app/                          ← Nuxt 4 app directory (runtime shell, NOT FSD)
│   ├── app.vue                   ← Root component
│   ├── pages/                    ← Thin routing shells (see "Thin page pattern")
│   │   ├── index.vue
│   │   └── products/
│   │       ├── index.vue
│   │       └── [id].vue
│   ├── layouts/                  ← Nuxt layouts
│   │   └── default.vue
│   ├── plugins/                  ← Nuxt plugins
│   ├── middleware/               ← Nuxt route middleware
│   └── composables/              ← Nuxt global composables
├── src/                          ← FSD root — all sliced layers live here
│   ├── pages/                    ← FSD page slices (full implementations)
│   │   ├── product-detail/
│   │   │   ├── ui/
│   │   │   │   └── ProductDetailPage.vue
│   │   │   ├── model/
│   │   │   │   └── useProductDetail.ts
│   │   │   └── index.ts
│   │   └── (user-settings)/      ← Slice group: only page slices, no shared code
│   │       ├── change-email/     ← page slice
│   │       │   ├── ui/
│   │       │   ├── model/
│   │       │   └── index.ts
│   │       └── two-fa/
│   ├── widgets/                  ← FSD widgets layer
│   │   ├── user-settings-shell/  ← page layout / shared chrome
│   │   │   ├── ui/
│   │   │   ├── model/
│   │   │   └── index.ts
│   │   └── product-card/
│   │       ├── ui/
│   │       │   └── ProductCard.vue
│   │       ├── model/
│   │       │   └── useProductCard.ts
│   │       └── index.ts
│   ├── features/                 ← FSD features layer
│   │   └── add-to-cart/
│   │       ├── ui/
│   │       │   └── AddToCartButton.vue
│   │       ├── model/
│   │       │   └── useAddToCart.ts
│   │       ├── api/
│   │       │   └── mutations.ts
│   │       └── index.ts
│   ├── entities/                 ← FSD entities layer
│   │   └── product/
│   │       ├── ui/
│   │       │   └── ProductPreview.vue
│   │       ├── model/
│   │       │   ├── types.ts
│   │       │   └── schema.ts
│   │       ├── api/
│   │       │   └── queries.ts
│   │       └── index.ts
│   └── shared/                   ← FSD shared layer
│       ├── ui/
│       │   ├── UiButton.vue
│       │   └── UiModal.vue
│       ├── lib/
│       │   ├── format-date.ts
│       │   └── cn.ts
│       ├── api/
│       │   └── client.ts
│       └── config/
│           └── constants.ts
├── server/                       ← Nuxt server routes (outside FSD client layers)
├── public/                       ← Static assets
└── nuxt.config.ts
```

## TypeScript configuration

Since `src/` lives outside the Nuxt `app/` directory, you must explicitly include it in the TypeScript config:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  typescript: {
    tsConfig: {
      include: ['../src/**/*'],
    },
    sharedTsConfig: {
      include: ['../src/**/*'],
    },
    nodeTsConfig: {
      include: ['../src/**/*'],
    },
  },
})
```

This ensures all three Nuxt-generated tsconfigs (client, shared, server) include type-checking and auto-completion for FSD code in `src/`.

## Optional path aliases

You can optionally configure shorter aliases in `nuxt.config.ts`:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  alias: {
    '@shared': fileURLToPath(new URL('./src/shared', import.meta.url)),
    '@entities': fileURLToPath(new URL('./src/entities', import.meta.url)),
    '@features': fileURLToPath(new URL('./src/features', import.meta.url)),
    '@widgets': fileURLToPath(new URL('./src/widgets', import.meta.url)),
  },
})
```

Then use: `import { UiButton } from '@shared/ui'`. Both patterns are valid — `~~/src/` requires no config, aliases are shorter.

## Shared layer structure

The `shared` layer has **no slices** — only segments:

```
src/shared/
  ui/               ← Generic UI components: buttons, modals, inputs, cards
  lib/              ← Utility functions: formatDate, cn(), debounce
  api/              ← API client setup, interceptors, base fetch wrapper
  config/           ← App-wide constants, env helpers, route names
  types/            ← Shared TypeScript utility types
```

Each segment can have its own `index.ts` for cleaner imports.

## Migration strategy

For existing Nuxt projects adopting FSD with `src/` root:

| Code | Approach |
|------|----------|
| New modules | Always follow FSD in `src/` |
| Existing `app/components/` | Migrate to `src/shared/ui/` or appropriate slice `ui/` segment |
| Existing `app/composables/` | Classify into feature/entity `model/` in `src/` or keep in `app/composables/` if truly global |
| Existing `app/utils/` | Move to `src/shared/lib/` |
| Existing `app/stores/` | Move to entity/feature `model/` segments in `src/` |
| Existing fat pages | Split into thin shell in `app/pages/` + page slice in `src/pages/` |

Steps:
1. Create `src/` directory with FSD layer subdirectories
2. Add `typescript` includes (`tsConfig`, `sharedTsConfig`, `nodeTsConfig`) to `nuxt.config.ts`
3. Start all new code in FSD structure under `src/`
4. Migrate existing code slice-by-slice when touched
5. Convert fat pages to thin routing shells + `src/pages/` slices
6. Update imports to use `~~/src/` and public APIs
7. Remove old directories once empty
