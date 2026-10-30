---
name: composition-over-configuration
description: >
  Builds Vue UI from small composable parts instead of one component with
  boolean, `showX` or config-array props. Use when creating or editing Vue
  SFCs, compound components (dialogs, inputs, menus, cards, composers), page
  or layout chrome, or when a component is growing props that only decide
  whether a child renders. Do not use to refactor existing kitchen-sink
  components unless the user asks.
---

# Composition over configuration

New UI is assembled from small parts. The parent template is the variant: what you don't want, you don't render.

Background: [Composition Is All You Need](https://www.youtube.com/watch?v=4KvbVq3Eg5w). Slack's message composer is a provider plus parts (`Frame`, `Input`, `Footer`, `Actions`), not `<Composer :is-thread :is-editing :show-emoji />`.

This applies to new UI. Leave existing kitchen-sink components alone unless the user asks to split them.

## Structure props vs. value props

A prop is configuration when it changes the **structure** of a component, meaning which children exist:

- booleans that mount a child: `showHeader`, `hasIcon`, `withFooter`, `isThread`
- modes that pick a different tree: `standalone`, `layout="compact"` when it drops parts
- arrays that describe children: `:actions="[{ label, icon, onClick }]"`

Props that change a **value or look** are fine and stay props:

- data: `title`, `items`, `modelValue`
- state: `disabled`, `loading`, `open`, `required`
- visual variants of one element: `variant`, `size`, `tone` (for example via `cva`)

Test: if a prop's only job is to decide whether a child renders, delete it. The parent renders the child or leaves it out.

## Build from parts

Follow the compound components the project already has: Reka UI, shadcn-vue, or the project's own wrappers around them. Use the wrappers when they exist.

```vue
<DialogRoot v-model:open="open">
  <DialogTrigger as-child>
    <Button>Delete</Button>
  </DialogTrigger>
  <DialogPortal>
    <DialogOverlay />
    <DialogContent>
      <DialogTitle>{{ title }}</DialogTitle>
      <DialogDescription>{{ description }}</DialogDescription>
      <slot />
      <DialogClose as-child>
        <Button>Cancel</Button>
      </DialogClose>
    </DialogContent>
  </DialogPortal>
</DialogRoot>
```

A dialog opened from code omits `DialogTrigger`. Don't add `:show-trigger`.

Leave out optional parts only. Parts that accessibility needs stay: a dialog without a visible title still renders `DialogTitle` inside `VisuallyHidden`, and Reka warns in the console when it's missing.

Inputs work the same way: `InputGroup` + `InputGroupInput` + `InputGroupAddon`, not `<TextField :has-icon :has-prefix :has-error />`.

Parts used only inside the module that owns them are imported by relative path. Parts that outside code uses are exported from that module's top-level public entry point (its `index.ts`), and outside code imports only from there. Don't add nested barrel files just to group a component's parts. Callers assemble the parts where they're used.

### Slot or part?

- **Slot:** free content in one spot of one component, like a dialog body.
- **Exported part:** the piece has its own props, moves around between variants, or shares state with its siblings.

A `v-if="$slots.footer"` around a footer wrapper is fine. Once the footer needs its own props or differs between callers, make it a part.

## Shared state: a root and its parts

A direct parent-child contract stays props down, events up. When several parts share state and the components between them would only relay it, the root provides it and each part injects it. Use `createInjectionState` from `@vueuse/core` and export an `OrThrow` consumer, so a part rendered outside its root fails loudly instead of silently getting `undefined`. Each root instance owns its state. Parts get readonly state and change it through actions.

```ts
import { createInjectionState } from '@vueuse/core'
import { readonly, shallowRef } from 'vue'

interface ComposerOptions {
  initial: string
  onSubmit: (text: string) => Promise<void>
}

const [useProvideComposer, useComposer] = createInjectionState(
  ({ initial, onSubmit }: ComposerOptions) => {
    const draft = shallowRef(initial)
    const setDraft = (text: string) => { draft.value = text }
    const submit = () => onSubmit(draft.value)
    return { draft: readonly(draft), setDraft, submit }
  },
)

export { useProvideComposer }

export function useComposerOrThrow() {
  const state = useComposer()
  if (!state)
    throw new Error('Composer parts must be rendered inside <ComposerRoot>')
  return state
}
```

The state lives in its own composable file next to the parts (`useComposer.ts`), not inside the root component. `ComposerRoot` calls `useProvideComposer(...)` in setup and renders `<slot />`. `ComposerInput` binds `:model-value="draft"` and `@update:model-value="setDraft"`, and `ComposerSubmit` calls `submit`. Both get them from `useComposerOrThrow()`, called in setup.

When your parts wrap Reka parts, read Reka's own state through its context (`injectDialogRootContext()` and similar) instead of copying `open` into a second provider.

What this buys you:

- A part outside the visual frame, like a send button in a header, reads the same state as long as it renders inside the root.
- Variants change what the root provides, not the parts. A new-message root and an edit root pass different sources into `useProvideComposer(...)` and share every part.

Flow state (a wizard `step`, the active tab) is state, not structure. It lives in the root or in `v-model`. Don't turn each step into a `showStepX` flag, and don't confuse it with chrome flags like `showSidebar`.

## Page and layout chrome

The same rule applies to pages and layouts. Don't grow this:

```vue
<SettingsPage
  :show-sidebar="isDesktop"
  :standalone="!isSignedIn"
  :with-delete-link="isOwner"
/>
```

Assemble it instead:

```vue
<SettingsLayout>
  <SettingsSection :title="t('email')">
    <ChangeEmailForm />
  </SettingsSection>
  <DeleteAccountLink v-if="isOwner" />
</SettingsLayout>
```

The `v-if` moved to the place that knows why the link is there. A guest flow with no chrome renders `<ChangeEmailForm />` on its own. Don't add `:standalone`.

When desktop and mobile share one child tree (one `<NuxtPage>` or `<RouterView>`), hide chrome with CSS (`max-md:hidden`). Don't `v-if` two copies of the tree behind `isMobile`: switching remounts the children and drops their state, and with SSR the server can't know the viewport, so hydration breaks.

When desktop and mobile really need different trees, render both and hide one with CSS, or wrap the client-only one in `<ClientOnly>` in Nuxt. Never branch on `window` or a viewport composable in a template that renders on the server.

## Presets for repeated assemblies

A preset is a named component built from the parts (`ConfirmDialog`, `SearchInput`). Make one when it makes callers easier to read, even for a single use, and always once the same assembly shows up three or more times. A different variant gets its own preset. Don't add a flag to the first one. Keep the parts exported so a one-off can still assemble its own.

## Where parts live

Parts start next to the one place that uses them. Split a component into local parts when a part has its own responsibility: own props, own state or its own place in the variants. A small component that changes as one unit stays one SFC. Moving parts to a shared location waits for proven reuse in three or more places. Until then, two copies are fine.

## Self-check

- [ ] No new prop whose only job is to mount or unmount a child
- [ ] No config array describing children the parent could render directly
- [ ] The parent template leaves out the parts it doesn't need
- [ ] Optional chrome is a sibling or wrapper component, not a flag on a mega-component
- [ ] Accessibility parts like `DialogTitle` are still rendered, visually hidden if needed
- [ ] Direct parent-child contracts use props and events, not injection
- [ ] Parts that share state get readonly state and actions from a root through an `OrThrow` consumer
- [ ] No template branches on `window` or viewport during SSR
- [ ] Visual variants (`variant`, `size`) are still plain props
- [ ] Existing kitchen-sink components are untouched unless the user asked
