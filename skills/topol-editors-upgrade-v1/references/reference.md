# Reference: v3 API → `@topol.io/editor*` 1.x (Email Editor v4)

Source of truth: the 1.x package READMEs on npm and the exported types (`IEmailConfig`, `IEmailCallbacks`, `IEmailEditorInstance`, `IEmailOptions`). Public guide: <https://docs.topol.io/email-editor/guide/npm-v1-migration.html>.

## Version map

| Line                                                             | npm tag           | Editor runtime | API shape                                                                                   |
| ---------------------------------------------------------------- | ----------------- | -------------- | ------------------------------------------------------------------------------------------- |
| `0.3.0`                                                          | `latest`          | v3             | `TopolPlugin.init({ id, authorize, callbacks })`, positional callback args                  |
| `1.0.0-alpha.1`–`.9` (editor-react up to `.10`)                  | none (superseded) | v3             | same, plus `EmailEditor` alias, multilanguage methods, `LandingPageEditor`                  |
| `1.0.0-alpha.10` (editor-react `.11`); `2.0.0-alpha.1`–`.3`      | none (superseded) | v4             | `EmailEditor.init({ config, ...callbacks })` → instance, object payloads; old names aliased |
| `1.0.0-alpha.11`+ (editor-react `1.0.0-alpha.12`+)               | `alpha`           | v4             | same shape; `entityId`, one type naming scheme, aliases removed, Svelte 5                   |

"Before" below means the v3 API (`0.3.0` and `1.0.0-alpha.1`–`.9`). "After" means the current 1.x, from `1.0.0-alpha.11` (editor-react `1.0.0-alpha.12`).

`TopolPlugin` (core), the core default export and `TopolEditor` (wrappers) no longer exist. The wrappers re-export the core factories as `CoreEmailEditor` / `CoreLandingPageEditor`.

## Coming from 0.3.0

These differences only affect apps on `0.3.0`; the early 1.x alphas already had them:

- **`onTestSend` `email` is `string | string[]`.** A handler that passes it straight to an API expecting one address needs a decision (send to all, or take the first). Ask; don't guess.
- **Container ids are gone.** 0.3.0 rendered into `#editor` (React) and `#topol-editor-id` (Vue, Svelte); the early 1.x alphas used `#topol-email-editor-id`. The wrappers now render a plain `div` without an id or `position: absolute`. CSS or DOM code targeting those ids must change.
- **Vue payload types** are imported from the package root, not from `./TopolEditor.vue`, under their new names (type table below).
- **`LandingPageEditor` is new** in core, React, and Vue. Adding it is optional; see `landing-page-editor.md`.
- **Multilanguage** callbacks and methods did not exist in 0.3.0. They are available with the shapes below.
- **Svelte:** the 0.3 component cannot load under Svelte 5 and the 1.x component cannot load under Svelte 4, so the Svelte major decides the path (step 2 of the skill).

## Coming from `1.0.0-alpha.10` (editor-react `.11`) or `2.0.0-alpha.x`

These apps already use `{ config, ...callbacks }`, the instance API and object payloads. Only this list applies; the rename tables further down give the details.

Breaking:

- **Removed names.** `TopolPlugin`, the core default export, the wrappers' `TopolEditor` component and `TopolPlugin` re-export. Import `EmailEditor` (or `CoreEmailEditor` from a wrapper).
- **Renamed types**, no aliases: see the type table.
- **Svelte wrapper.** Requires Svelte 5. `on:` events, `event.detail`, the `callbacks` prop, `on:ready` and `ISvelteEmailEvents` are removed; every callback is a prop, `onReady` replaces `on:ready`, `ISvelteOptions` is `ISvelteEmailOptions`. See *Svelte before / after*.
- **`onEditorError` `source`.** `templateLoad` is now `load`. The React and Vue `LandingPageEditor` report a failed `init()` / `render()` as `integration` instead of `init`.
- **Removed Email Editor options**: `googleApiKey`, `imageEditorV2`, `api.IMAGE_EDITOR_UPLOAD`, `api.TEST_EMAIL`, `enableCountdown`, the `"v4"` `lambdaStage`, and the `raw`, `custom-raw` and `apiBlock` keys of `contentBlocks` (the editor rejects the whole `contentBlocks` object when one is present; use `customBlocks[].disabled` for custom blocks).
- **`onSavedSectionSave`** hands over `{ section: { definition, translation? } }` without `id` or `name`; the host assigns them. `onSavedSectionRemove` / `onSavedSectionEdit` type `id` as `number`.
- **API blocks** (`apiBlocks`) need a `name`.
- **Landing Page Editor** wrappers, instance and option types changed; see `landing-page-editor.md`.

Deprecated, still working — rename now (table in *Config table* and *Callback table*): `authorize.userId`, `onEdittedWithoutSaveChanged`, `updateTestingEmailAddresses`, `onBlocksAndStructuresVisibilityChange`, `toggleBlocksAndStructuresVisibility()`, `toggleControlPanel()`, `hideControlPanel`, `showControlPanelExpand`, `productMergetagTags`, `dynamicMergetags`, `api.PREMADE_TEMPLATES_KEYWORDS`, and the dashed `defaultTemplateSettings` keys.

New and optional: `save()` rejects with `err.code === "SAVE_IN_PROGRESS"` while another save runs; `close()` is typed as an alias of `destroy()`; `onEditorClose` may return a promise; `disableShortcuts`, `saveAndCloseButton`, `clickTrackingOptOut`, `startWithOpenElementsPanel`, the `"jinja"` `emailServiceProvider`; `updateApiAuthorizationHeader` removes a header given `null`.

## Core before / after

```ts
// Before (v3 API)
import TopolPlugin from "@topol.io/editor"; // or { EmailEditor }
await TopolPlugin.init(
  {
    id: "#editor",
    authorize: { apiKey: "key", userId: "user-123" },
    savedBlocks: blocks,
    callbacks: {
      onSave(json, html, mutations, syncedSections) {
        persist(json, html);
      },
      onError(type, message, responseBody) {
        report(type, message);
      },
      onClose() {
        leave();
      },
    },
  },
  { stage: "production" }
);
TopolPlugin.save("en");
TopolPlugin.setSavedBlocks(blocks);
TopolPlugin.destroy();
```

```ts
// After (1.x, Email Editor v4)
import { EmailEditor, type IEmailConfig } from "@topol.io/editor";
const config: IEmailConfig = {
  authorize: { apiKey: "key", entityId: "user-123" },
  savedSections: blocks,
};
const editor = await EmailEditor.init(
  {
    config,
    async onSave({ json, html, mutations, syncedSections }) {
      await persist(json, html);
    },
    onEditorError({ source, message, detail }) {
      report(source, message);
    },
    onEditorClose() {
      leave();
    },
  },
  { stage: "production" } // or { loaderUrl: "https://.../loader/build.js" }
);
await editor.render("#editor"); // element must exist and have a height
await editor.save({ langCode: "en" });
await editor.setSavedSections(blocks);
await editor.destroy(); // on unmount, not immediately after render
```

`init` rejects for: missing `config.authorize`, an invalid `entityId`, `cloud.enabled` without `getNonce`, an empty `loaderUrl`, or a non-browser environment (call it after mount).

## React before / after

```tsx
// Before (v3 API)
import TopolEditor from "@topol.io/editor-react";           // 0.3.0; early 1.x: { EmailEditor }
import { TopolPlugin } from "@topol.io/editor-react";
<TopolEditor options={options} stage="production"
  onSave={(json, html) => persist(json, html)}
  onError={(type, message) => report(type, message)} />
<button onClick={() => TopolPlugin.save()} />
```

```tsx
// After (1.x, Email Editor v4)
import { useRef } from "react";
import { EmailEditor, type IEmailEditorRef, type IReactEmailOptions } from "@topol.io/editor-react";
const ref = useRef<IEmailEditorRef>(null);
const options: IReactEmailOptions = { authorize: { apiKey: "key", entityId: "user-123" } };
<EmailEditor ref={ref} options={options} stage="production"
  onSave={async ({ json, html }) => { await persist(json, html); }}
  onEditorError={({ source, message }) => report(source, message)}
  onReady={(editor) => { /* instance available */ }}
  className="editor" style={{ height: "80vh" }} />
<button onClick={() => ref.current?.getEditor()?.save()} />
```

`IReactEmailOptions` is `IEmailConfig`. Props (`IEmailEditorProps`) accept every `IEmailCallbacks` member plus `stage`, `loaderUrl`, `onReady`, `className`, `style`. StrictMode is supported. There is no default export and no `TopolEditor`.

## Vue before / after

```vue
<!-- Before (v3 API) -->
<script setup lang="ts">
import { TopolEditor, TopolPlugin, type ITopolOptions } from "@topol.io/editor-vue";
const options = { authorize: { apiKey: "key", userId: "user-123" } };
</script>
<template>
  <TopolEditor
    :options="options"
    @onSave="({ json, html }) => persist(json, html)"
    @onError="report"
  />
  <button @click="TopolPlugin.save()" />
</template>
```

```vue
<!-- After (1.x, Email Editor v4) -->
<script setup lang="ts">
import { ref } from "vue";
import { EmailEditor, type IEmailCallbacks, type IVueEmailOptions } from "@topol.io/editor-vue";
const options: IVueEmailOptions = { authorize: { apiKey: "key", entityId: "user-123" } };
const callbacks: IEmailCallbacks = {
  async onSave({ json, html }) {
    await persist(json, html);
  }, // awaited by the editor
};
const editorRef = ref<InstanceType<typeof EmailEditor>>();
</script>
<template>
  <EmailEditor
    ref="editorRef"
    :options="options"
    :callbacks="callbacks"
    @on-editor-error="({ source, message }) => report(source, message)"
    @ready="(editor) => {}"
  />
  <button @click="editorRef?.getEditor()?.save()" />
</template>
```

Events (`@on-save`, `@on-editor-error`, ...) carry the exact core payload as the single argument; their return values are not awaited. Use the `callbacks` prop for `onSave` / `onSaveAndClose` / `onEditorClose` that work asynchronously and for `getNonce` (function callback only, never an event). When both a callback and an event listener exist, the callback runs first and its result is returned. `@on-save-and-close` is registered only when supplied; without it the runtime's default save-and-close works. Template ref exposes `getEditor()`.

## Svelte before / after

Requires Svelte 5.

```svelte
<!-- Before: 0.3.0 and the v3-API alphas -->
<script lang="ts">
  import { TopolEditor, TopolPlugin, type ITopolOptions } from "@topol.io/editor-svelte";
  const options: ITopolOptions = { authorize: { apiKey: "key", userId: "user-123" } };
</script>

<TopolEditor
  {options}
  on:onSave={(e) => persist(e.detail.json, e.detail.html)}
  on:onError={(e) => report(e.detail)}
/>
<button on:click={() => TopolPlugin.save()} />
```

```svelte
<!-- Before: 1.0.0-alpha.10 / 2.0.0-alpha.x -->
<EmailEditor
  bind:this={component}
  {options}
  callbacks={{ getNonce, onSave }}
  on:onEditorError={(e) => report(e.detail.source, e.detail.message)}
  on:ready={(e) => { editor = e.detail; }}
/>
```

```svelte
<!-- After (current 1.x, Svelte 5) -->
<script lang="ts">
  import { EmailEditor, type ISvelteEmailOptions, type IEmailEditorInstance, type IEmailSavePayload } from "@topol.io/editor-svelte";
  const options: ISvelteEmailOptions = { authorize: { apiKey: "key", entityId: "user-123" } };
  let editor: IEmailEditorInstance | null = null;
  async function onSave({ json, html }: IEmailSavePayload) {
    await persist(json, html); // awaited by the editor
  }
</script>

<EmailEditor
  {options}
  {onSave}
  {getNonce}
  onEditorError={({ source, message }) => report(source, message)}
  onReady={(instance) => (editor = instance)}
  style="height: 80vh"
/>
<button onclick={() => editor?.save()}>Save</button>
```

Every callback is a prop receiving the payload object directly (no `event.detail`); a returned promise (`onSave`, `onSaveAndClose`, `onEditorClose`, `getNonce`) is awaited. `ISvelteEmailOptions` is `IEmailConfig`; `IEmailEditorProps` types all props. `bind:this` + `getEditor()` still returns the instance. `class` / `style` props set the container.

## Wrapper semantics (all frameworks)

- Configuration and the **presence** of handlers are captured on mount. Adding/removing a handler or changing config needs a remount (React/Vue `key`, Svelte `{#key}`). Existing handler functions read their latest closure without a remount.
- `ready` / `onReady` fires after `render` mounted the iframe. `onInit` fires after successful authentication. Wait for `onInit` before calling methods that need an authenticated editor.
- Authentication failures arrive through `onEditorError`; the editor exports no methods afterwards. Destroy and recreate after fixing credentials.
- Each component owns one instance and destroys it on unmount, including while `init` is still pending.
- Wrapper containers render `width: 100%; height: 100vh` by default, with no id and no `position: absolute`. Override with React `style`/`className`, Vue attribute fallthrough (`class`/`style` land on the root `div`), or the Svelte `class`/`style` props.
- Wrapper-side failures (loader, init, render) are reported through `onEditorError` with `source: "integration"`, or `console.error` when no handler exists.

## Method table (`TopolPlugin.x` → `instance.x`)

All current methods return `Promise<void>` unless noted.

| Before (v3 API)                                                               | After (1.x, v4)                                                                                                      |
| ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `init(options, { stage })` → `Promise<boolean>`                               | `init({ config, ...callbacks }, { stage?, loaderUrl? })` → `Promise<IEmailEditorInstance>`; then `render(container)` |
| `save(lang?)`                                                                 | `save({ langCode? })` — resolves after render/persist/host `onSave` settle; rejects with `code: "SAVE_IN_PROGRESS"` while another save runs |
| `load(json)`                                                                  | `load(json)` — resolves once applied                                                                                 |
| `setSavedBlocks(blocks)`                                                      | `setSavedSections(sections)`                                                                                         |
| `refreshSyncedRows()`                                                         | `refreshSyncedSections()`                                                                                            |
| `getMutations()` → `void` (result via `onGetMutations`)                       | `getMutations()` → `Promise<{ code, primary }[]>` (also fires `onGetMutations`)                                      |
| `updateOptions(partial)`                                                      | `updateOptions(partial)` — `authorize` excluded                                                                      |
| `createLanguage / deleteLanguage / selectLanguage / setPrimaryLanguage(lang)` | same names, argument is `code`                                                                                       |
| `toggleBlocksAndStructuresVisibility()`                                       | `toggleHiddenElementsVisibility()` (old name deprecated)                                                             |
| `toggleControlPanel()`                                                        | `toggleSelectionPanel()` (old name deprecated)                                                                       |
| `updateApiAuthorizationHeader(header)`                                        | same; an object is merged, a `null` value removes that header                                                        |
| —                                                                             | `translateLanguage(code, sourceCode?)`, `translateAllLanguages()`                                                    |
| `setMergeTags` (typed on `window.TopolPlugin` only)                           | `setMergeTags(tags)` on the instance                                                                                 |
| `destroy()`                                                                   | `destroy()` → `Promise<void>`; `close()` is an alias                                                                 |

Unchanged names: `togglePreview`, `togglePreviewSize`, `toggleDarkMode`, `chooseFile`, `undo`, `redo`, `setPreviewHTML`, `createNotification`, `setActiveMembers`, `changeEmailToMobile`, `changeEmailToDesktop`, `updateCustomBlockContent`, `refreshComments`, `openPremadeTemplatesSelection`, `setTemplateName`, `toggleChatAI`, `toggleAutosaves`, `toggleComments`, `updateTemplate(json, { skipSnapshot })`.

## Callback table

Every current callback takes zero arguments or one object. Wrapper props and events deliver the same object.

| Before (v3 API)                                                                     | After (1.x, v4)                                                                       |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `onSave(json, html, mutations?, syncedSections?)`                                   | `onSave({ json, html, mutations, syncedSections })` → `void \| Promise<void>`         |
| `onSaveAndClose(json, html, mutations?, syncedSections?)`                           | `onSaveAndClose({ json, html, mutations, syncedSections })` → `void \| Promise<void>` |
| `onTestSend(email, json, html)`                                                     | `onTestSend({ email, json, html })`, `email` is `string \| string[]`                  |
| `onBannerClick(json, html)`                                                         | `onBannerClick({ json, html })`                                                       |
| `onOpenFileManager()` / `onLoaded()` / `onInit()`                                   | unchanged                                                                             |
| `onBlockSave(block)`                                                                | `onSavedSectionSave({ section })`, `section` is `{ definition, translation? }`        |
| `onBlockRemove(id)`                                                                 | `onSavedSectionRemove({ id })`                                                        |
| `onBlockEdit(id)`                                                                   | `onSavedSectionEdit({ id })`                                                          |
| `onUndoChange(count)` / `onRedoChange(count)`                                       | `onUndoChange({ count })` / `onRedoChange({ count })`                                 |
| `onPreview(html)`                                                                   | `onPreview({ html })`                                                                 |
| `onAlert(notification)`                                                             | `onAlert({ notification })`                                                           |
| `onClose()`                                                                         | `onEditorClose()` → `void \| Promise<void>`                                           |
| `onEdittedWithoutSaveChanged(bool)`                                                 | `onUnsavedChangesChange({ hasUnsavedChanges })`                                       |
| `onOpenCustomBlockDialog(content)`                                                  | `onOpenCustomBlockDialog({ block })`                                                  |
| `onTemplateRename(title)`                                                           | `onTemplateRename({ name })`                                                          |
| `updateTestingEmailAddresses(emails)`                                               | `onTestingEmailsChange({ emails })`                                                   |
| `onError(type, message, responseBody?)`                                             | `onEditorError({ source, message, detail? })`                                         |
| `onLanguageCreated / onLanguageDeleted / onPrimaryLanguageChanged(lang, mutations)` | `({ code, mutations })`, mutations are `{ code, primary }[]`                          |
| `onLanguageSelected(lang)`                                                          | `onLanguageSelected({ code })`                                                        |
| `onGetMutations(mutations)`                                                         | `onGetMutations({ mutations })`                                                       |

New in 1.x (v4): `onPreviewClose()`, `onImageDelete({ items })` with items `{ name, type, path, url?, key }`, `onHiddenElementsVisibilityChange({ visible })`, `onTemplateUpdated()`, the nonce authentication callback `getNonce`, and the Cloud callbacks `onCloudSave`, `onCloudTemplateChange`.

Deprecated object-payload names from `1.0.0-alpha.10` that still fire next to their replacement and log a warning: `onEdittedWithoutSaveChanged({ value })`, `updateTestingEmailAddresses({ emails })`, `onBlocksAndStructuresVisibilityChange({ value })` (fires only on the reveal on device switch, always `false`). Rename them and read the new payload key.

`onEditorError` `source` is `EditorErrorSource`: `integration`, `init`, `authorize`, `nonceAuth`, `network`, `load` (was `templateLoad`), `updateTemplate`, `save`, `testSend`, `cloudTemplates`, `cloudSync`.

Reserved keys: `onClose` and `onError` are stripped before reaching the editor. Any handler under those names is dead code.

v3 wrapper-specific payload differences that also change: Vue events already carried objects for `onSave`, `onSaveAndClose`, `onTestSend`, `onBannerClick` (keep the shape, mutations now use `code`), but passed bare values for `onUndoChange`, `onRedoChange`, `onPreview`, `onAlert`, `onEdittedWithoutSaveChanged`, `onOpenCustomBlockDialog`, `onTemplateRename`, `updateTestingEmailAddresses`, `onBlock*` (now objects as above). Vue language events carried `{ lang, mutations }` → now `{ code, mutations }`; the Vue `onError` event `{ type, message, responseBody }` → `onEditorError` `{ source, message, detail }`. Svelte events are gone altogether: every `on:onX={(e) => f(e.detail)}` becomes the prop `onX={(payload) => f(payload)}` with the payloads of this table.

## Config table (`IEmailConfig`)

| Before (`ITopolOptions`, v3 API)                                                             | After (`IEmailConfig`, 1.x v4)                                                                                         |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `id`                                                                                         | removed — pass the container to `render()` / the wrapper owns it                                                       |
| `callbacks: { ... }`                                                                         | removed — callbacks sit beside `config` in the factory props                                                           |
| `authorize.userId: string \| number`                                                         | `authorize.entityId`: non-negative integer, or string matching `[A-Za-z0-9_-]+` (`userId` is a deprecated alias; `entityId` wins when both are set) |
| `savedBlocks: ISavedBlock[] \| boolean`                                                      | `savedSections: ISavedSection[] \| boolean \| null`                                                                    |
| `premadeBlocks`, `premadeBlocksNoOverwrite`                                                  | `premadeSections: false \| sections \| { blocks, override }`                                                           |
| `syncedRowsEnable`                                                                           | `syncedSectionsEnabled`                                                                                                |
| `topBarOptions: string[]` (controls to show)                                                 | `hideTopbarControls: string[]` (controls to hide)                                                                      |
| `defaultTemplateSettings.langs[].key`                                                        | `defaultTemplateSettings.langs[].code`                                                                                 |
| `api.SAVE`, `api.LOAD`, `api.PREVIEW`                                                        | removed — persist through `onSave`, load with `editor.load()`                                                          |
| `api.AUTOSAVE`, `api.GET_AUTOSAVE`, `api.SYNCED_ROWS`                                        | removed — autosaves are served from `api.AUTOSAVES`                                                                    |
| `api.TEST_EMAIL`, `api.IMAGE_EDITOR_UPLOAD`                                                  | removed — test sends arrive in `onTestSend`                                                                            |
| `api.SAVED_BLOCKS`                                                                           | `api.SAVED_SECTIONS`                                                                                                   |
| `googleApiKey`, `imageEditorV2`, `lambdaStage: "v4"`                                         | removed                                                                                                                |
| `contentBlocks.raw`, `contentBlocks["custom-raw"]`, `contentBlocks.apiBlock`                 | removed — the whole `contentBlocks` object is rejected when one is present; use `customBlocks[].disabled`              |
| —                                                                                            | `api.GENERATE_TEXT`, `GENERATE_PREHEADER`, `GENERATE_TRANSLATION`, `PRODUCT_CATEGORIES`, `PREMADE_TEMPLATES`, `PREMADE_TEMPLATE_CATEGORIES`, `PREMADE_TEMPLATE_KEYWORDS` |
| —                                                                                            | `cloud: { enabled, templateId?, features? }`                                                                           |
| —                                                                                            | `permissions`, `enableSubjectLine`, `chatAIOnSegment`, `aiLabeledElements`, `languageMergeTag`, `enableFileManagerInGif`, `googlePromotionalAnnotation`, `ecm`, `disableShortcuts`, `saveAndCloseButton`, `clickTrackingOptOut`, `startWithOpenElementsPanel`, `countdown`, loop/rating content blocks, image-editor options |

Renamed with the old name still accepted (deprecated; the new name wins when both are set):

| Deprecated                                                  | Use                                                   |
| ----------------------------------------------------------- | ----------------------------------------------------- |
| `authorize.userId`                                          | `authorize.entityId`                                  |
| `hideControlPanel`                                          | `hideSelectionPanel`                                  |
| `showControlPanelExpand`                                    | `hideSelectionPanelExpand` (inverted)                 |
| `productMergetagTags`                                       | `productMergeTags`                                    |
| `apiBlocks.*.dynamicMergetags`                              | `dynamicMergeTags`                                    |
| `api.PREMADE_TEMPLATES_KEYWORDS`                            | `api.PREMADE_TEMPLATE_KEYWORDS`                       |
| `defaultTemplateSettings.blockBackgroundColor`              | `sectionBackgroundColor`                              |
| `defaultTemplateSettings["structure-margin"]`               | `sectionPadding`                                      |
| `defaultTemplateSettings["line-height"]` / `["stack-columns"]` | `lineHeight` / `stackColumns`                      |

A removed or misspelled option left in the config is ignored without an error in plain JavaScript; the typecheck is what finds it.

### `textOverride` keys

| Before (v3 API) | After (1.x, v4) |
| --- | --- |
| `saved_blocks*` | `saved_sections*` |
| `premade_blocks` | `premade_sections` |
| `save_as_block` | `save_as_section` |
| `synced_rows` | `synced_sections` |
| `*_synced_row*` | `*_synced_section*` |

The keys are plain strings, so the typecheck does not find a stale one; grep for them.

### Changed defaults

| Option | v3 | v4 |
| --- | --- | --- |
| `smartMergeTags.enabled` | `false` | `true` |
| `mobileFirstEnabled` | `false` | `true` |
| `premadeTemplates` | `false` | `true` |
| Selection-panel expand button | hidden (`showControlPanelExpand: false`) | shown; hide it with `hideSelectionPanelExpand: true` |

The selection panel also starts hidden when the iframe is narrower than 1024 px. Keeping or pinning these is a product decision; ask the user.

## Type table

The old names are removed, not aliased. Data going into or out of an editor is `IEmail*` / `ILandingPage*`; types of the editor object are `IEmailEditor*` / `ILandingPageEditor*`.

| Old name                                                   | New name                                                        |
| ---------------------------------------------------------- | --------------------------------------------------------------- |
| `ITopolOptions` as the config object (0.3, early alphas)   | `IEmailConfig`                                                  |
| `ITopolOptions` as `{ config, ...callbacks }` (alpha.10)   | `IEmailOptions`                                                 |
| `ISaveData`                                                | `IEmailSavePayload` (`mutations` are `{ code, primary }[]`)     |
| `IErrorData`                                               | `IEmailErrorPayload` (`{ source, message, detail? }`)           |
| `ISendTestData`                                            | `IEmailTestSendPayload`                                         |
| `ISavedBlock`                                              | `ISavedSection`                                                 |
| `ITopolPlugin`                                             | `IEmailEditorInstance`                                          |
| React `EmailEditorProps`, `TopolEditorProps`               | `IEmailEditorProps`                                             |
| React `IReactTopolOptions`                                 | `IReactEmailOptions`                                            |
| React `LandingPageEditorProps` / `LandingPageEditorRef`    | `ILandingPageEditorProps` / `ILandingPageEditorRef`             |
| Vue `IVueOptions`                                          | `IVueEmailOptions`                                              |
| Svelte `ISvelteOptions`                                    | `ISvelteEmailOptions`                                           |
| Svelte `ISvelteEmailEvents`                                | removed — `IEmailEditorProps` types the callback props          |
| `ICloudCallbacks["getNonce"]`                              | `INonceAuthCallbacks["getNonce"]`                               |
| `TopolSection`                                             | removed — template JSON is `unknown`; type the part you read    |

Other exported types: `IEmailCallbacks`, `IEmailEditorPluginOptions` (`{ stage?, loaderUrl? }`), `IAuthorize`, `EditorErrorSource`, `ISavedSectionDraft`, `IPremadeSections`, `ILanguageMutation`, `ILanguageChangePayload`, `IEmailDeletedItem`, `INonceAuthCallbacks`, `ICloudCallbacks`, `ICloudOptions`, `ICloudFeatures`, `ICountdownOptions`, `EmailEditorShortcut`. Every wrapper re-exports them.

## Loader and environments

- Default loader: `https://v4.email-assets.topol.io/loader/build.js`. The v3 loader is incompatible and rejected.
- `stage`: `"production"` (default), `"dev"`, `"staging"` (both resolve to the release loader unless the build-time override is set), or a number selecting a PR preview.
- `loaderUrl` (init option / wrapper prop) selects a local or preview loader explicitly and never falls back. The upstream local loader uses the host page origin for the iframe, so serve or proxy the editor at that origin.
- Build-time overrides for the package build: `VITE_TOPOL_V4_URL`, `VITE_TOPOL_V4_DEV_URL`, `VITE_TOPOL_V4_STAGING_URL`. `VITE_TOPOL_URL` and the other v3 variables are ignored.
- One loader URL per page. A second `init` with a different URL rejects: "Editor v4 supports one loader URL per page."
- Let `init` insert the script. A host-inserted `<script src=…loader/build.js>` or a foreign `window.TopolEmailEditor` rejects with "Cannot initialize from a host-provided loader with unknown lifecycle." An already loaded production runtime is reused.
- Numeric-stage loader failures fall back to production only before any runtime has initialized.

## Backend and security policy

Apps coming from the v3 API that host their own `api.*` endpoints or set a Content Security Policy need changes outside the frontend code. Flag them; do not guess at the backend.

- CORS on the `api.*` endpoints must allow `https://v4.email-assets.topol.io` instead of the v3 origin.
- Autosaves use one URL: `GET AUTOSAVES` (list), `POST AUTOSAVES` (create, was `POST AUTOSAVE`), `GET AUTOSAVES/{key}` (read one, was `GET GET_AUTOSAVE/{key}`). Bodies are unchanged.
- `SAVED_SECTIONS` listing requests can include `type=folder`; return only folders then.
- CSP / Permissions-Policy: replace `v3.email-assets.topol.io` with `v4.email-assets.topol.io` in `script-src`, `frame-src` and feature allowlists.

Details: <https://docs.topol.io/email-editor/guide/v4-migration.html#backend-and-security-policy>.

Deploy order: the autosave route change must be live before the upgraded frontend ships. Autosave params and bodies (`key`, `hostname`, `templateId`, `uuid`, `definition`, `current_user_id`) are unchanged, and any 2xx is accepted. Before the first autosave of a session, v4 lists autosaves, reads the newest, and may create one extra "initial state" autosave. Autosave is skipped while an explicit save is running.

## Behaviour changes

Runtime differences between Email Editor v3 and v4 that no rename reveals. Report them to the user of an app coming from the v3 API.

- **Awaited saves.** A Promise returned from `onSave` / `onSaveAndClose` keeps the saving indicator up until it settles. A rejection shows a saving error and reports `onEditorError({ source: "save" })`. After 30 s the editor stops waiting and shows an "unconfirmed" notice without `onEditorError`.
- **Unsaved flag.** It clears only after `onSave` settles and only if the template did not change meanwhile; v3 cleared it after rendering.
- **Save & Close without `onSaveAndClose`.** v4 calls `onSave` and then closes the editor (removes the iframe); v3 did nothing.
- **Window-bar close without `onEditorClose`.** v4 closes the editor itself; v3 needed the callback. If `onEditorClose` rejects, the editor also closes itself.
- **Failed init.** Method calls made before the editor is ready are queued and resolve after initialization. If authentication or init fails, the host gets `onEditorError({ source: "authorize" })` and then `onEditorError({ source: "init" })`, and the queued calls never settle. Recover with `destroy()` and a new `init()`.
- **Hostname for authorization.** Taken from the parent page through the iframe bridge, so a strict `Referrer-Policy` no longer breaks authorization.

## Nonce authentication

Passing `getNonce` switches authentication from the public key alone to short-lived `nc_` nonces that the host's backend creates with the Secret key. It is standalone: there is no separate option, the presence of the callback enables it, and templates, autosaves, images, comments, and sections keep using the host's `api.*` endpoints and callbacks. It works for the Email Editor and the Landing Page Editor. Topol Cloud is built on top of it.

```ts
const editor = await EmailEditor.init({
  config: { authorize: { apiKey: "your-public-key", entityId: "user-123" } },
  async getNonce() {
    // Runs on the host page with first-party session cookies. Your backend creates the
    // nonce for this entityId with the Secret key. sk_ secrets never reach the browser.
    const response = await fetch("/api/topol/nonce", { method: "POST" });
    if (!response.ok) throw new Error("Could not get a Topol nonce");
    return (await response.json()).nonce; // a fresh nc_ token
  },
});
```

Rules:

- `getNonce` is a top-level callback in core, a prop in React and Svelte, and a `callbacks` member in Vue (never an event). Its type is `INonceAuthCallbacks["getNonce"]`.
- The nonce endpoint is the host's responsibility: authenticate the session, derive the user, bind the nonce to the same `entityId` as `authorize.entityId`, expire it within an hour. The backend either signs the nonce itself (HMAC-SHA256 with the Secret key) or asks Topol's `POST /auth/nonce` for one; the signing code and request format are in <https://docs.topol.io/email-editor/guide/nonce-authentication.html> and <https://docs.topol.io/landing-page-editor/guide/nonce-authentication.html>. Scaffold the endpoint as a marked TODO unless the user asks for it.
- The editor refreshes the nonce before it expires and retries once on a rejected nonce. A first nonce that cannot be obtained reports `onEditorError` with `source: "authorize"`; a later refresh failure reports `source: "nonceAuth"` while the editor keeps working on the last valid nonce.

## Topol Cloud

Cloud (Email Editor) moves templates, autosaves, images, comments, sections, multilanguage, and test-send into Topol's backend. It runs on nonce authentication, so set up `getNonce` as above first.

```ts
const editor = await EmailEditor.init({
  config: {
    authorize: { apiKey: "your-public-key", entityId: "user-123" },
    cloud: { enabled: true, templateId: 123, features: { images: false } },
  },
  getNonce, // required; see Nonce authentication
  onCloudSave({ templateId, name, langCode }) {},
  onCloudTemplateChange({ type, templateId, name, folderId }) {}, // type: rename | delete | duplicate | move
});
```

Rules:

- `cloud.enabled` needs `getNonce`; `init` rejects without it.
- Each feature can be switched back to external behaviour with `features.<name>: false`. Cloud-active features ignore their external `api` endpoints and callbacks and warn about them. `onSave` and `onTestSend` do not fire for Cloud-owned persistence and test-send; `onSaveAndClose`, lifecycle, error, and notification callbacks still do. `instance.save()` works in either mode.
- `cloud.templateId` opens a stored template; omitting it opens the Cloud template browser. `config.templateId` is ignored in Cloud mode. A new template's id arrives via `onCloudSave`.

## Leftover grep (must return nothing after migration)

```sh
grep -rnE "\bTopolPlugin\b|\bTopolEditor\b|\bid: *['\"]#|callbacks: *\{|savedBlocks|setSavedBlocks|premadeBlocks|syncedRowsEnable|refreshSyncedRows|topBarOptions|onBlockSave|onBlockRemove|onBlockEdit|SAVED_BLOCKS|SYNCED_ROWS|VITE_TOPOL_URL\b|topol-email-editor-id|topol-editor-id|topol-landing-page-editor-id|\bonError\(|\bonClose\(|\bon:on[A-Z]|\bon:ready\b|templateLoad|\b(ITopolOptions|ITopolPlugin|ISaveData|IErrorData|ISendTestData|ISavedBlock|ISvelteOptions|ISvelteEmailEvents|IVueOptions|IReactTopolOptions|TopolEditorProps|EmailEditorProps|LandingPageEditorProps|LandingPageEditorRef|TopolSection)\b" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git \
  --exclude=package-lock.json --exclude=yarn.lock --exclude=pnpm-lock.yaml
```

Expected false positives to read and dismiss: `callbacks: {` on a line that builds the Vue `callbacks` prop, and the app's own `onError(` / `onClose(` functions that are not Topol handlers.

Stale `textOverride` keys (apps coming from the v3 API):

```sh
grep -rnE "saved_blocks|premade_blocks|save_as_block|synced_row" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git
```

Deprecated names — still working, but the upgrade should leave none behind:

```sh
grep -rnE "onEdittedWithoutSaveChanged|updateTestingEmailAddresses|onBlocksAndStructuresVisibilityChange|toggleBlocksAndStructuresVisibility|toggleControlPanel|hideControlPanel|showControlPanelExpand|productMergetagTags|dynamicMergetags|PREMADE_TEMPLATES_KEYWORDS|blockBackgroundColor|structure-margin|line-height|stack-columns|userId" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git \
  --exclude=package-lock.json --exclude=yarn.lock --exclude=pnpm-lock.yaml
```

Only hits inside a Topol config or handler count: `userId` matters in an `authorize` object (and in the backend's nonce payload), `line-height` and friends in `defaultTemplateSettings`.
