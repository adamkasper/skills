# Common mistakes

Read this when you review FSD code or a placement looks wrong. Each entry pairs the mistake with the fix.

### 1. Putting FSD layers inside `app/`
**Wrong**: `app/features/`, `app/entities/`, `app/widgets/`, `app/shared/`.
**Right**: FSD layers live in `src/`. Only `app/pages/` (thin shells), `app/layouts/`, `app/plugins/`, `app/middleware/`, `app/composables/` go in `app/`.

### 2. Fat page routes
**Wrong**: Putting full page implementations in `app/pages/products/[id].vue` (200+ lines).
**Right**: `app/pages/` files are thin routing shells (5–25 lines) that import from `src/pages/`.

### 3. Premature extraction
**Wrong**: Creating `src/features/fancy-button` because "it might be reused."
**Right**: Keep in `src/pages/` until 3 actual usages prove the need.

### 4. Wrong layer
**Wrong**: Putting `useProductSearch` (standalone logic) in a widget.
**Right**: Ask "can this logic work without THIS specific UI?" — yes means Feature.

### 5. Missing public API
**Wrong**: `import { x } from '~~/src/features/auth/model/internal'`
**Right**: All exports go through `index.ts`.

### 6. Upward imports
**Wrong**: `src/entities/user` importing from `src/features/auth`.
**Right**: Dependency flows only downward — entities never reference features.

### 7. Same-layer cross-imports
**Wrong**: `src/features/profile` importing from `src/features/auth`.
**Right**: Extract shared concept to `entities/` or use events.

### 8. Business logic in shared
**Wrong**: `src/shared/lib/useAuth.ts`, `src/shared/lib/useProductSearch.ts`.
**Right**: Shared is for generic utilities only — zero business logic.

### 9. Relying on Nuxt auto-imports for FSD layers
**Wrong**: Expecting `useAuth()` to auto-resolve from `src/features/auth/model/`.
**Right**: Use explicit imports via `~~/src/features/auth` public API.

### 10. Applying FSD outside `src/`
**Wrong**: Organizing `app/composables/`, `app/middleware/`, or `server/` into FSD layers.
**Right**: FSD lives exclusively in `src/`. Everything else (`app/`, `server/`) follows Nuxt conventions.

### 11. Putting shared code in a slice group
**Wrong**: `src/pages/(user-settings)/index.ts`, `ui/`, `model/`, or `_layout/` next to the page slices.
**Right**: The group folder contains only page slices. Shared chrome is a widget (`src/widgets/user-settings-shell`). Isolated groups like `(blog)` have no chrome at all.
