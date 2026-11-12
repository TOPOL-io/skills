# Landing Page Editor in 1.x

The Landing Page Editor arrived during the 1.x alphas and changed along the way.
Which part of this file applies depends on where the app starts:

- **Coming from 0.3.0:** the Landing Page Editor is **new functionality**, not
  part of the upgrade. Only add it when the user asks for it; *Core API* and
  *Framework wrappers* below show the current shape.
- **Landing Page Editor code written against `1.0.0-alpha.1`–`.8`** (editor-react
  up to `.9`): its callbacks took positional arguments. They now take one object;
  see *Callbacks*. Everything in the next bullet applies too.
- **Landing Page Editor code written against any alpha up to `1.0.0-alpha.10`**
  (editor-react `.11`): the wrappers, the instance and the option types changed;
  see *Changes since `1.0.0-alpha.10`*.

`init()` resolves to an **instance** that you then render and control, the same
model as the 1.x Email Editor.

## Core API

```ts
import { LandingPageEditor } from "@topol.io/editor";

const editor = await LandingPageEditor.init({
  config: {
    authorize: { apiKey: "your-key", entityId: "user-123" },
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
await editor.load(templateJson); // optional: load a template
await editor.save(); // trigger onSave; resolves once it settled
await editor.destroy(); // clean up
```

Editor settings live under **`config`** (`ILandingPageConfig`); callbacks sit at
the **top level** next to it. The second argument of `init()` is
`{ loaderUrl? }` (`ILandingPageEditorPluginOptions`).

## Callbacks

Every callback takes no arguments or one object:

| Callback | `1.0.0-alpha.1`–`.8` | Now |
| --- | --- | --- |
| `onSave` / `onSaveAndClose` | `(json, html)` | `({ json, html, syncedSections? })`, may return a Promise the editor awaits |
| `onImageDelete` | `(items)` | `({ items })` |
| `onUndoChange` / `onRedoChange` | `(count)` | `({ count })` |
| `onPreview` | `(html)` | `({ html })` |
| `onAlert` | `(notification)` | `({ notification })` |
| `onEdittedWithoutSaveChanged` | `(value)` | **`onUnsavedChangesChange({ hasUnsavedChanges })`**; the old name still fires with `({ value })` but is deprecated |
| `onOpenCustomBlockDialog` | `(block)` | `({ block })` |
| `onTemplateRename` | `(name)` | `({ name })` |
| `onClose` | `()` | **`onEditorClose()`**, may return a Promise |
| `onError` | `(type, message, responseBody?)` | **`onEditorError({ source, message, detail? })`** |
| `onBannerClick`, `onBlockSave`, `onBlockRemove`, `onBlockEdit` | typed | removed (the runtime never emitted them) |

Unchanged: `onOpenFileManager()`, `onLoaded()`, `onInit()`. New: `onPreviewClose()`,
`getNonce()` (nonce authentication, see `reference.md`).
`onClose` and `onError` are reserved by the loader and never reach the editor.
`onEditorClose` is not called when the host tears the editor down with `destroy()`.

Payload types: `ILandingPageSavePayload`, `ILandingPageDeletedItem`,
`ILandingPageErrorPayload` (`source` is `EditorErrorSource`, shared with the Email
Editor).

## Changes since `1.0.0-alpha.10`

Breaking:

- **Wrapper container.** The React and Vue components no longer render
  `#topol-landing-page-editor-id` with `position: absolute`; the container is a
  plain `div` of `width: 100%; height: 100vh`. CSS or tests that target the id,
  and layouts that relied on the absolute positioning, must set their own
  (`className` / `style` in React, `class` / `style` in Vue).
- **Wrapper ref.** `load()` and `save()` return promises instead of `void`;
  `getEditor()` is new and returns the instance (not a reactive proxy in Vue).
- **Renamed React types.** `LandingPageEditorRef` → `ILandingPageEditorRef`,
  `LandingPageEditorProps` → `ILandingPageEditorProps`.
- **Error sources.** A failed `init()` / `render()` in a wrapper is reported as
  `source: "integration"` (was `"init"`); `"templateLoad"` is now `"load"`. A
  failed `render()` destroys the instance first, so `getEditor()` returns `null`.
- **`TopolSection` is removed** from all packages. Templates are typed
  `unknown`; define a type for the part of the JSON the app reads.
- **Instance methods all return promises**, including `load()` and `save()`.
  `save()` rejects with `err.code === "SAVE_IN_PROGRESS"` while another save runs.
- **Removed options**: `premadeTemplatesOptions.showDelete`,
  `theme.colors["primary-dark"]`, the `raw` and `custom-raw` `contentBlocks` keys,
  `googleApiKey`, `chatAI`, `disableExtendedComparators`, `helpdesk`,
  `imageEditor`, `premadeBlocks` and `savedBlocks` (now `premadeSections` and
  `savedSections`), and the `api` keys `SAVE`, `AUTOSAVE`, `GET_AUTOSAVE`, `LOAD`,
  `PREVIEW`, `PRODUCTS`, `SYNCED_SECTIONS`, `SAVED_BLOCKS` (now `SAVED_SECTIONS`)
  and `TEST_EMAIL`. The notification field `extarnalUse` is gone.
- **Loader URL.** `init()` loads `https://v1.page-assets.topol.io/loader/build.js`
  instead of `…/topol-lpe.js`. A Content Security Policy that lists the script by
  full path needs the new one.

Deprecated, still working — rename now: `authorize.userId` → `entityId`,
`onEdittedWithoutSaveChanged` → `onUnsavedChangesChange`,
`defaultTemplateSettings["template-color"]` → `templateColor`,
`api.PREMADE_TEMPLATES_KEYWORDS` → `api.PREMADE_TEMPLATE_KEYWORDS`.

New and optional: nonce authentication (`getNonce`), `loaderUrl`, the wrappers'
`onReady` / `@ready`, the Vue `callbacks` prop, and the instance methods
`togglePreview`, `setPreviewSize`, `undo`, `redo`, `chooseFile`,
`refreshSyncedSections`, `refreshComments`, `openPremadeTemplatesSelection`,
`setTemplateName`, `updateOptions`, `toggleSelectionPanel`, `toggleAutosaves`,
`toggleComments`, `setActiveMembers`, `updateCustomBlockContent`,
`updateApiAuthorizationHeader`, `createNotification` and `close()` (alias of
`destroy()`).

## Framework wrappers

React and Vue ship components with the same shape as `EmailEditor`. Both take the
**config** object as `options` (no `config` wrapper), expose the callbacks with
the same object payloads, and render a plain container of
`width: 100%; height: 100vh` without an id. Wrappers only forward the callbacks
you provide: without `onSaveAndClose` / `onEditorClose` the runtime falls back to
`onSave` and closes itself.

React — callbacks are props; imperative methods come from the ref:

```tsx
import { useRef } from "react";
import { LandingPageEditor, type ILandingPageEditorRef, type IReactLandingPageOptions } from "@topol.io/editor-react";

const options: IReactLandingPageOptions = { authorize: { apiKey, entityId } };
const ref = useRef<ILandingPageEditorRef>(null);

<LandingPageEditor
  ref={ref}
  options={options}
  onSave={async ({ json, html }) => { await save(json, html); }}
  onReady={(editor) => { /* instance available */ }}
  style={{ height: "80vh" }}
/>;
// await ref.current?.load(templateJson);
// await ref.current?.save();
// ref.current?.getEditor()?.togglePreview();
```

Vue — events are notifications; handlers the editor must await, and `getNonce`,
go in the `callbacks` prop:

```vue
<script setup lang="ts">
import { LandingPageEditor, type ILandingPageCallbacks, type IVueLandingPageOptions } from "@topol.io/editor-vue";
const options: IVueLandingPageOptions = { authorize: { apiKey, entityId } };
const callbacks: ILandingPageCallbacks = {
  async onSave({ json, html }) {
    await save(json, html);
  },
};
</script>

<template>
  <LandingPageEditor :options="options" :callbacks="callbacks" @on-editor-error="report" @ready="(editor) => {}" />
</template>
```

The Vue template ref exposes `load()`, `save()` and `getEditor()`. Both wrappers
accept `loaderUrl` (`loader-url` in Vue).

Svelte has no landing-page component — use the core API from `onMount`, and call
`editor.destroy()` when the component is destroyed.

## Constraints

- `stage` is accepted but currently ignored — the Landing Page Editor always
  loads the production bundle unless `loaderUrl` is given. One loader URL per
  page. Do not promise dev/staging environments.
- The container element must exist in the DOM before `render()` is called.
- The config is validated at runtime, at init and on `updateOptions()`: a
  wrongly-typed option stops the editor from starting and is reported through
  `onEditorError` with `source: "init"`. Unknown keys are ignored.
- `onLoaded` fires after every `load()`; load the first template from `onInit`
  or after `render()`, never from `onLoaded`.
- Template JSON is not interchangeable with email template JSON.
