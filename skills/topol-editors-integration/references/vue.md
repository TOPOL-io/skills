# Vue 3 — `@topol.io/editor-vue`

Peer dep: `vue` `^3.2.25`. Ships both editors as components. Vue 2 apps must use
the core package (`vanilla.md`).

```bash
npm install @topol.io/editor-vue@<resolved 1.x alpha version>
```

## Email Editor

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { EmailEditor, type IEmailCallbacks, type IVueEmailOptions } from '@topol.io/editor-vue';

const options: IVueEmailOptions = {
  authorize: {
    apiKey: import.meta.env.VITE_TOPOL_API_KEY,
    userId: currentUser.id,   // integer or [A-Za-z0-9_-]+
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
  are not awaited. Put `onSave` / `onSaveAndClose` that persist asynchronously,
  and `getNonce`, in the `callbacks` prop. If both exist, the callback runs first
  and its result goes back to the editor.
- `@on-save-and-close` is registered only when supplied; without it (and without
  `callbacks.onSaveAndClose`) the runtime's default save-and-close works.
- The template ref exposes `getEditor()` (instance or `null`); `@ready` delivers
  the instance once the iframe has rendered. `onInit` means authenticated.
- The component renders a plain `div` (`width: 100%; height: 100vh`, no id);
  `class` / `style` attributes fall through to it.
- Props: `options`, `callbacks`, `stage`, `loaderUrl` (leave the last two unset in
  production).

## Landing Page Editor

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { LandingPageEditor, type IVueLandingPageOptions } from '@topol.io/editor-vue';

const editor = ref<InstanceType<typeof LandingPageEditor> | null>(null);

const options: IVueLandingPageOptions = {
  authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, userId: currentUser.id },
};

const load = () => editor.value?.load(templateJson);
const save = () => editor.value?.save();
</script>

<template>
  <button @click="load">Load</button>
  <button @click="save">Save</button>
  <LandingPageEditor
    ref="editor"
    :options="options"
    @on-save="({ json, html }) => { /* TODO: persist */ }"
    @on-editor-error="({ source, message }) => console.error(source, message)"
  />
</template>
```

- `options` is the LPE **config** object only; the component wraps it as
  `{ config, ...callbacks }` and calls `render()` itself.
- `load(template)` and `save()` are exposed via the template ref. Unmount to
  destroy.
- Container is `#topol-landing-page-editor-id` at `position: absolute;
  width: 100%; height: 100vh`.
- LPE has no `onTestSend`, no `updateTestingEmailAddresses`, and no language events.

## Lifecycle

Both components init in `onMounted` and destroy on unmount (including while init
is still pending). Configuration and which handlers are present are captured on
mount; remount (`:key`) to change them, or call
`editorRef.value?.getEditor()?.updateOptions(partial)` for Email Editor config.

## Deprecated

`TopolEditor` is a deprecated alias of `EmailEditor` with the same props and
events. Use `EmailEditor`.
