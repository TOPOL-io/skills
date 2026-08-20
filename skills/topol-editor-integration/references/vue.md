# Vue 3 — `@topol.io/editor-vue`

Peer dep: `vue` `^3.2.25`. Ships both editors as components. Vue 2 apps must use
the core package (`vanilla.md`).

```bash
npm install @topol.io/editor-vue@<resolved-v1-version>
```

## Email Editor

```vue
<script setup lang="ts">
import { EmailEditor, type IVueEmailOptions, type ISaveData } from '@topol.io/editor-vue';

const options: IVueEmailOptions = {
  authorize: {
    apiKey: import.meta.env.VITE_TOPOL_API_KEY,
    userId: currentUser.id,
  },
};

const handleSave = ({ json, html, mutations, syncedSections }: ISaveData) => {
  // TODO: persist the template — application decision
};
</script>

<template>
  <EmailEditor
    :options="options"
    @onSave="handleSave"
    @onError="({ type, message, responseBody }) => console.error(type, message, responseBody)"
  />
</template>
```

- Events are declared in camelCase (`onSave`, `onSaveAndClose`, `onTestSend`, …).
  `@onSave` and the kebab form `@on-save` both bind.
- **Every event payload is a single object**, not positional args:
  `onSave`/`onSaveAndClose` → `{ json, html, mutations, syncedSections }`,
  `onTestSend` → `{ email, json, html }`, `onError` → `{ type, message, responseBody }`,
  `onBannerClick` → `{ json, html }`, the language events → `{ lang, mutations }`.
  Simple events pass their value directly (`onBlockRemove` → `blockId`,
  `onUndoChange` → `count`, `onAlert` → notification).
- `options` is `Omit<ITopolOptions, 'id' | 'callbacks'>`; the component renders
  `<div id="topol-email-editor-id">` at `position: absolute; width: 100%; height: 100vh`.
- `stage` prop defaults to `'production'`.

## Landing Page Editor

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { LandingPageEditor, type IVueLandingPageOptions, type ISaveData } from '@topol.io/editor-vue';

const editor = ref<InstanceType<typeof LandingPageEditor> | null>(null);

const options: IVueLandingPageOptions = {
  authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, userId: currentUser.id },
};

const load = () => editor.value?.load(templateJson);
const save = () => editor.value?.save();
const handleSave = ({ json, html }: ISaveData) => {/* TODO: persist */};
</script>

<template>
  <button @click="load">Load</button>
  <button @click="save">Save</button>
  <LandingPageEditor ref="editor" :options="options" @onSave="handleSave" />
</template>
```

- `options` is the LPE **config** object only; the component wraps it as
  `{ config, ...callbacks }` and calls `render()` itself.
- `load(template)` and `save()` are exposed via `defineExpose` — reach them
  through a template ref. Unmount to destroy.
- Container id is `topol-landing-page-editor-id`.
- LPE has no `onTestSend`, no `updateTestingEmailAddresses`, and no language events.
- The LPE `onSave` / `onSaveAndClose` payload is typed `ISaveData` but only ever
  contains `{ json, html }` — `mutations` and `syncedSections` are email-only and
  arrive `undefined`. Destructure just the two fields.

## Lifecycle

Both components init in `onMounted` and destroy in `onBeforeUnmount`. Changing
`options` after mount does not re-initialize — use
`CoreEmailEditor.updateOptions(partial)` (re-exported from this package) for
runtime config changes. Render only one editor component at a time: each mounts
an absolutely-positioned full-viewport container.

## Deprecated

The `TopolEditor` component still exists as an alias and logs a console
deprecation warning on mount. Use `EmailEditor`.
