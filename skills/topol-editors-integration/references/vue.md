# Vue 3 — `@topol.io/editor-vue`

Peer dep: `vue` `^3.2.25`. Ships both editors as components. Vue 2 apps must use
the core package (`vanilla.md`).

```bash
npm install @topol.io/editor-vue@<resolved 1.x version>
```

## Email Editor

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { EmailEditor, type IEmailCallbacks, type IVueEmailOptions } from '@topol.io/editor-vue';

const options: IVueEmailOptions = {
  authorize: {
    apiKey: import.meta.env.VITE_TOPOL_API_KEY,
    entityId: currentUser.id,   // integer or [A-Za-z0-9_-]+
  },
};

// Handlers whose result the editor must await go in the `callbacks` prop.
const callbacks: IEmailCallbacks = {
  async onSave({ json, html, mutations, syncedSections }) {
    // TODO: persist the template — application decision. A rejection shows a save error.
  },
};

const editorRef = ref<InstanceType<typeof EmailEditor>>();
const save = () => editorRef.value?.getEditor()?.save();
</script>

<template>
  <button @click="save">Save</button>
  <EmailEditor
    ref="editorRef"
    :options="options"
    :callbacks="callbacks"
    @on-editor-error="({ source, message, detail }) => console.error(source, message, detail)"
    @ready="(editor) => { /* iframe rendered; instance available */ }"
  />
</template>
```

- `options` is the config (`IEmailConfig`).
- **`callbacks` prop vs events.** Events (`@on-save`, `@on-editor-error`, …)
  carry the exact core object payload but are notifications: their return values
  are not awaited. Put `onSave` / `onSaveAndClose` / `onEditorClose` that work
  asynchronously, and `getNonce`, in the `callbacks` prop. If both exist, the
  callback runs first and its result goes back to the editor.
- `@on-save-and-close` is registered only when supplied; without it (and without
  `callbacks.onSaveAndClose`) the runtime's default save-and-close works.
- The template ref exposes `getEditor()` (instance or `null`); `@ready` delivers
  the instance once the iframe has rendered. `onInit` means authenticated.
- The component renders a plain `div` (`width: 100%; height: 100vh`, no id);
  `class` / `style` attributes fall through to it.
- Props: `options`, `callbacks`, `stage`, `loader-url` (leave the last two unset
  in production).

## Landing Page Editor

Same shape as `EmailEditor`.

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { LandingPageEditor, type ILandingPageCallbacks, type IVueLandingPageOptions } from '@topol.io/editor-vue';

const editorRef = ref<InstanceType<typeof LandingPageEditor> | null>(null);

const options: IVueLandingPageOptions = {
  authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, entityId: currentUser.id },
};

const callbacks: ILandingPageCallbacks = {
  async onSave({ json, html }) {
    // TODO: persist — awaited by the editor
  },
};

// Editor ready: load the first template here, not in @on-loaded.
const onInit = () => editorRef.value?.load(templateJson);
const save = () => editorRef.value?.save();
</script>

<template>
  <button @click="save">Save</button>
  <LandingPageEditor
    ref="editorRef"
    :options="options"
    :callbacks="callbacks"
    @on-init="onInit"
    @on-editor-error="({ source, message }) => console.error(source, message)"
  />
</template>
```

- `options` is the LPE **config** object only (`ILandingPageConfig`); the
  component wraps it as `{ config, ...callbacks }` and calls `render()` itself.
- The `callbacks` prop (`ILandingPageCallbacks`) works as on `EmailEditor`: use it
  for `getNonce` and for async `onSave` / `onSaveAndClose` / `onEditorClose`.
  Events remain for notifications.
- The template ref exposes `load(template)` and `save()`, both returning
  promises, and `getEditor()` for every other instance method. `@ready` delivers
  the instance. Unmount to destroy.
- The container is a plain `div` (`width: 100%; height: 100vh`, no id);
  `class` / `style` fall through to it. Several editors can share a page.
- Props: `options`, `callbacks`, `loader-url`; `stage` is accepted and ignored.
- A failed `init()` / `render()` is reported as `onEditorError` with
  `source: 'integration'`.
- LPE has no `onTestSend`, no testing-email callbacks, and no language events.

## Lifecycle

Both components init in `onMounted` and destroy on unmount (including while init
is still pending). Configuration and which handlers are present are captured on
mount; remount (`:key`) to change them, or call
`editorRef.value?.getEditor()?.updateOptions(partial)`.

## Names that no longer exist

There is no `TopolEditor` component and no `TopolPlugin` re-export; import
`EmailEditor` (the core factories are `CoreEmailEditor` /
`CoreLandingPageEditor`). `IVueOptions` is now `IVueEmailOptions`.
