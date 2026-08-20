# Adding the Landing Page Editor (new in 1.x)

This is **new functionality**, not part of the 0.x → 1.x upgrade. Only do this
when the user asks for it.

The Landing Page Editor differs from the Email Editor in one important way:
`init()` resolves to an **instance** that you then render and control.

## Core API

```ts
import { LandingPageEditor } from '@topol.io/editor';

const editor = await LandingPageEditor.init({
  config: {
    authorize: { apiKey: 'your-key', userId: 'user-123' },
    // …the rest of the editor config: api, theme, language, savedBlocks, …
  },
  onSave(json, html) { /* … */ },
  onLoaded() { /* … */ },
});

editor.render('#landing-page-editor');  // mount into an existing container
editor.load(templateJson);              // optional: load a template
editor.save();                          // trigger the save callback
editor.destroy();                       // clean up
```

Note the shape: editor settings live under **`config`**, while callbacks sit at
the **top level** of the options object — unlike the Email Editor, where
callbacks live under `options.callbacks`.

## Callbacks

`onSave`, `onSaveAndClose`, `onLoaded`, `onInit`, `onClose`, `onBannerClick`,
`onOpenFileManager`, `onBlockSave(block: TopolSection)`,
`onBlockRemove(id: number | string)`, `onBlockEdit(id: number | string)`,
`onUndoChange`, `onRedoChange`, `onPreview`, `onAlert`,
`onEdittedWithoutSaveChanged`, `onOpenCustomBlockDialog`, `onTemplateRename`,
`onError(type, message, responseBody?)`.

Block ids are `number | string` here (the Email Editor uses `number`).

## Framework wrappers

React and Vue ship components that own the lifecycle for you. Both take the
**config** object as `options` (no `config` wrapper, no `callbacks` key) and
expose the callbacks as props/events. Both render into
`#topol-landing-page-editor-id`.

React — imperative methods come from the ref:

```tsx
import { useRef } from 'react';
import { LandingPageEditor, type LandingPageEditorRef, type IReactLandingPageOptions } from '@topol.io/editor-react';

const options: IReactLandingPageOptions = { authorize: { apiKey, userId } };
const ref = useRef<LandingPageEditorRef>(null);

<LandingPageEditor ref={ref} options={options} onSave={(json, html) => save(json, html)} />
// ref.current?.load(templateJson);
// ref.current?.save();
```

Vue:

```vue
<script setup lang="ts">
import { LandingPageEditor, type IVueLandingPageOptions } from '@topol.io/editor-vue';
const options: IVueLandingPageOptions = { authorize: { apiKey, userId } };
</script>

<template>
  <LandingPageEditor :options="options" @on-save="handleSave" />
</template>
```

Svelte has no landing-page component — use the core API from `onMount`, and call
`editor.destroy()` in the cleanup function.

## Constraints

- `stage` is accepted but currently ignored — the Landing Page Editor always
  loads the production bundle. Do not promise dev/staging environments.
- The container element must exist in the DOM before `render()` is called.
- Template JSON is not interchangeable with email template JSON.
