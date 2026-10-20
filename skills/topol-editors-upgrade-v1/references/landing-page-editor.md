# Landing Page Editor in 1.x

The Email Editor v4 move does not change the Landing Page Editor. Two situations
still need this file:

- **Coming from 0.3.0:** the Landing Page Editor is **new functionality**, not
  part of the upgrade. Only add it when the user asks for it.
- **Landing Page Editor code written against `1.0.0-alpha.1`–`.8`** (editor-react
  up to `.9`): its callbacks took positional arguments. They now take one object;
  see *Callbacks* below.

`init()` resolves to an **instance** that you then render and control, the same
model as the 1.x Email Editor.

## Core API

```ts
import { LandingPageEditor } from "@topol.io/editor";

const editor = await LandingPageEditor.init({
  config: {
    authorize: { apiKey: "your-key", userId: "user-123" },
    // …the rest of the editor config
  },
  async onSave({ json, html }) {
    await persist(json, html); // awaited; a rejection shows a save error
  },
  onEditorError({ source, message, detail }) {
    report(source, message);
  },
});

await editor.render("#landing-page-editor"); // mount into an existing container
editor.load(templateJson); // optional: load a template
editor.save(); // trigger onSave
await editor.destroy(); // clean up
```

Editor settings live under **`config`**; callbacks sit at the **top level** next
to it.

## Callbacks

Every callback takes no arguments or one object:

| Callback | `1.0.0-alpha.1`–`.8` | Now |
| --- | --- | --- |
| `onSave` / `onSaveAndClose` | `(json, html)` | `({ json, html })`, may return a Promise the editor awaits |
| `onImageDelete` | `(items)` | `({ items })` |
| `onUndoChange` / `onRedoChange` | `(count)` | `({ count })` |
| `onPreview` | `(html)` | `({ html })` |
| `onAlert` | `(notification)` | `({ notification })` |
| `onEdittedWithoutSaveChanged` | `(value)` | `({ value })` |
| `onOpenCustomBlockDialog` | `(block)` | `({ block })` |
| `onTemplateRename` | `(name)` | `({ name })` |
| `onClose` | `()` | **`onEditorClose()`**, may return a Promise |
| `onError` | `(type, message, responseBody?)` | **`onEditorError({ source, message, detail? })`** |
| `onBannerClick`, `onBlockSave`, `onBlockRemove`, `onBlockEdit` | typed | removed (the runtime never emitted them) |

Unchanged: `onOpenFileManager()`, `onLoaded()`, `onInit()`. New: `onPreviewClose()`.
`onClose` and `onError` are reserved by the loader and never reach the editor.

Payload types: `ILandingPageSavePayload`, `ILandingPageDeletedItem`,
`ILandingPageErrorPayload` (`source` is `"authorize" | "network" | "templateLoad" | "init" | string`).

## Framework wrappers

React and Vue ship components that own the lifecycle for you. Both take the
**config** object as `options` (no `config` wrapper, no `callbacks` key), expose the
callbacks as props/events with the same object payloads, and render into
`#topol-landing-page-editor-id` (`position: absolute; width: 100%; height: 100vh`).
Wrappers only forward the callbacks you provide: without `onSaveAndClose` /
`onEditorClose` the runtime falls back to `onSave` and closes itself.

React — imperative methods come from the ref:

```tsx
import { useRef } from "react";
import { LandingPageEditor, type LandingPageEditorRef, type IReactLandingPageOptions } from "@topol.io/editor-react";

const options: IReactLandingPageOptions = { authorize: { apiKey, userId } };
const ref = useRef<LandingPageEditorRef>(null);

<LandingPageEditor ref={ref} options={options} onSave={async ({ json, html }) => { await save(json, html); }} />;
// ref.current?.load(templateJson);
// ref.current?.save();
```

Vue:

```vue
<script setup lang="ts">
import { LandingPageEditor, type IVueLandingPageOptions } from "@topol.io/editor-vue";
const options: IVueLandingPageOptions = { authorize: { apiKey, userId } };
</script>

<template>
  <LandingPageEditor :options="options" @on-save="({ json, html }) => save(json, html)" @on-editor-error="report" />
</template>
```

Svelte has no landing-page component — use the core API from `onMount`, and call
`editor.destroy()` in `onDestroy`.

## Constraints

- `stage` is accepted but currently ignored — the Landing Page Editor always
  loads the production bundle. Do not promise dev/staging environments.
- The container element must exist in the DOM before `render()` is called.
- The config is validated at runtime: a misspelled or wrongly-typed option is
  rejected, not ignored.
- Template JSON is not interchangeable with email template JSON.
