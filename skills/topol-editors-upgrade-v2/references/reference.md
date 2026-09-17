# Reference: `@topol.io/editor*` 1.x → 2.x

Source of truth: the 2.x package READMEs on npm and the exported types (`IEmailConfig`, `IEmailCallbacks`, `IEmailEditorInstance`, `ITopolOptions`).

## Version map

| Line            | npm tag                      | Editor runtime | API shape                                                                  |
| --------------- | ---------------------------- | -------------- | -------------------------------------------------------------------------- |
| `0.3.0`         | `latest`                     | v3             | `TopolPlugin.init({ id, authorize, callbacks })`, positional callback args |
| `1.0.0-alpha.x` | none (pin the exact version) | v3             | same, plus `EmailEditor` alias, multilanguage methods, `LandingPageEditor` |
| `2.x`           | `alpha` (`2.0.0-alpha.x`)    | v4             | `EmailEditor.init({ config, ...callbacks })` → instance, object payloads   |

`TopolPlugin` (core) and `TopolEditor` (wrappers) still exist in 2.x as naming aliases of the new API. They are not v3 adapters: code using them must still be migrated.

## Core before / after

```ts
// 1.x
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
// 2.x
import { EmailEditor, type IEmailConfig } from "@topol.io/editor";
const config: IEmailConfig = {
  authorize: { apiKey: "key", userId: "user-123" },
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

`init` rejects for: missing `config.authorize`, an invalid `userId`, `cloud.enabled` without `getNonce`, an empty `loaderUrl`, or a non-browser environment (call it after mount).

## React before / after

```tsx
// 1.x
import TopolEditor from "@topol.io/editor-react";           // or { EmailEditor }
import { TopolPlugin } from "@topol.io/editor-react";
<TopolEditor options={options} stage="production"
  onSave={(json, html) => persist(json, html)}
  onError={(type, message) => report(type, message)} />
<button onClick={() => TopolPlugin.save()} />
```

```tsx
// 2.x
import { useRef } from "react";
import { EmailEditor, type IEmailEditorRef, type IReactEmailOptions } from "@topol.io/editor-react";
const ref = useRef<IEmailEditorRef>(null);
const options: IReactEmailOptions = { authorize: { apiKey: "key", userId: "user-123" } };
<EmailEditor ref={ref} options={options} stage="production"
  onSave={async ({ json, html }) => { await persist(json, html); }}
  onEditorError={({ source, message }) => report(source, message)}
  onReady={(editor) => { /* instance available */ }}
  className="editor" style={{ height: "80vh" }} />
<button onClick={() => ref.current?.getEditor()?.save()} />
```

`IReactEmailOptions` is `IEmailConfig`. Props accept every `IEmailCallbacks` member plus `stage`, `loaderUrl`, `onReady`, `className`, `style`. StrictMode is supported. `TopolEditor` remains a deprecated alias with the same props.

## Vue before / after

```vue
<!-- 1.x -->
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
<!-- 2.x -->
<script setup lang="ts">
import { ref } from "vue";
import { EmailEditor, type IEmailCallbacks, type IVueEmailOptions } from "@topol.io/editor-vue";
const options: IVueEmailOptions = { authorize: { apiKey: "key", userId: "user-123" } };
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

Events (`@on-save`, `@on-editor-error`, ...) carry the exact core payload as the single argument; their return values are not awaited. Use the `callbacks` prop for `onSave` / `onSaveAndClose` that persist asynchronously and for `getNonce` (function callback only, never an event). When both a callback and an event listener exist, the callback runs first and its result is returned. `@on-save-and-close` is registered only when supplied; without it the runtime's default save-and-close works. Template ref exposes `getEditor()`.

## Svelte before / after

```svelte
<!-- 1.x -->
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
<!-- 2.x -->
<script lang="ts">
  import { EmailEditor, type ISvelteOptions, type IEmailCallbacks } from "@topol.io/editor-svelte";
  const options: ISvelteOptions = { authorize: { apiKey: "key", userId: "user-123" } };
  const callbacks: IEmailCallbacks = {
    async onSave({ json, html }) {
      await persist(json, html);
    },
  };
  let component: EmailEditor;
</script>

<EmailEditor
  bind:this={component}
  {options}
  {callbacks}
  on:onEditorError={(e) => report(e.detail.source, e.detail.message)}
  on:ready={(e) => {
    const editor = e.detail;
  }}
/>
<button on:click={() => component.getEditor()?.save()} />
```

`ISvelteOptions` is `IEmailConfig`. `ITopolOptions` from the Svelte package now refers to the core factory props, not the wrapper config. Event payloads arrive in `event.detail` with the exact v4 shape (1.x dispatched bare strings/arrays for some events, e.g. `onLanguageSelected`, `onGetMutations`). The component exposes `getEditor()`. `TopolEditor` remains as an alias of `EmailEditor`.

## Wrapper semantics (all frameworks)

- Configuration and the **presence** of handlers are captured on mount. Adding/removing a handler or changing config needs a remount (React/Vue `key`). Existing handler functions read their latest closure without a remount.
- `ready` / `onReady` fires after `render` mounted the iframe. `onInit` fires after successful authentication. Wait for `onInit` before calling methods that need an authenticated editor.
- Authentication failures arrive through `onEditorError`; the editor exports no methods afterwards. Destroy and recreate after fixing credentials.
- Each component owns one instance and destroys it on unmount, including while `init` is still pending.
- Wrapper containers render `width: 100%; height: 100vh` by default. Override with React `style`/`className` or Vue attribute fallthrough (`class`/`style` land on the root `div`); the Svelte component takes no attributes, so size its parent element.
- Wrapper-side failures (loader, init, render) are reported through `onEditorError` with `source: "integration"`, or `console.error` when no handler exists.

## Method table (`TopolPlugin.x` → `instance.x`)

All 2.x methods return `Promise<void>` unless noted.

| 1.x                                                                           | 2.x                                                                                                                  |
| ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `init(options, { stage })` → `Promise<boolean>`                               | `init({ config, ...callbacks }, { stage?, loaderUrl? })` → `Promise<IEmailEditorInstance>`; then `render(container)` |
| `save(lang?)`                                                                 | `save({ langCode? })` — resolves after render/persist/host `onSave` settle                                           |
| `load(json)`                                                                  | `load(json)` — resolves once applied                                                                                 |
| `setSavedBlocks(blocks)`                                                      | `setSavedSections(sections)`                                                                                         |
| `refreshSyncedRows()`                                                         | `refreshSyncedSections()`                                                                                            |
| `getMutations()` → `void` (result via `onGetMutations`)                       | `getMutations()` → `Promise<{ code, primary }[]>` (also fires `onGetMutations`)                                      |
| `updateOptions(partial)`                                                      | `updateOptions(partial)` — `authorize` excluded                                                                      |
| `createLanguage / deleteLanguage / selectLanguage / setPrimaryLanguage(lang)` | same names, argument is `code`                                                                                       |
| —                                                                             | `translateLanguage(code, sourceCode?)`, `translateAllLanguages()`                                                    |
| `setMergeTags` (typed on `window.TopolPlugin` only)                           | `setMergeTags(tags)` on the instance                                                                                 |
| `destroy()`                                                                   | `destroy()` → `Promise<void>`                                                                                        |

Unchanged names: `togglePreview`, `togglePreviewSize`, `toggleDarkMode`, `chooseFile`, `undo`, `redo`, `setPreviewHTML`, `createNotification`, `setActiveMembers`, `changeEmailToMobile`, `changeEmailToDesktop`, `toggleBlocksAndStructuresVisibility`, `updateCustomBlockContent`, `refreshComments`, `openPremadeTemplatesSelection`, `updateApiAuthorizationHeader`, `setTemplateName`, `toggleChatAI`, `toggleAutosaves`, `toggleComments`, `toggleControlPanel`, `updateTemplate(json, { skipSnapshot })`.

## Callback table

Every 2.x callback takes zero arguments or one object. Wrapper events deliver the same object.

| 1.x                                                                                 | 2.x                                                                                   |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `onSave(json, html, mutations, syncedSections)`                                     | `onSave({ json, html, mutations, syncedSections })` → `void \| Promise<void>`         |
| `onSaveAndClose(json, html, mutations, syncedSections)`                             | `onSaveAndClose({ json, html, mutations, syncedSections })` → `void \| Promise<void>` |
| `onTestSend(email, json, html)`                                                     | `onTestSend({ email, json, html })`                                                   |
| `onBannerClick(json, html)`                                                         | `onBannerClick({ json, html })`                                                       |
| `onOpenFileManager()` / `onLoaded()` / `onInit()`                                   | unchanged                                                                             |
| `onBlockSave(block)`                                                                | `onSavedSectionSave({ section })`                                                     |
| `onBlockRemove(id)`                                                                 | `onSavedSectionRemove({ id })`                                                        |
| `onBlockEdit(id)`                                                                   | `onSavedSectionEdit({ id })`                                                          |
| `onUndoChange(count)` / `onRedoChange(count)`                                       | `onUndoChange({ count })` / `onRedoChange({ count })`                                 |
| `onPreview(html)`                                                                   | `onPreview({ html })`                                                                 |
| `onAlert(notification)`                                                             | `onAlert({ notification })`                                                           |
| `onClose()`                                                                         | `onEditorClose()`                                                                     |
| `onEdittedWithoutSaveChanged(bool)`                                                 | `onEdittedWithoutSaveChanged({ value })` (upstream spelling kept)                     |
| `onOpenCustomBlockDialog(content)`                                                  | `onOpenCustomBlockDialog({ block })`                                                  |
| `onTemplateRename(title)`                                                           | `onTemplateRename({ name })`                                                          |
| `updateTestingEmailAddresses(emails)`                                               | `updateTestingEmailAddresses({ emails })`                                             |
| `onError(type, message, responseBody?)`                                             | `onEditorError({ source, message, detail? })`                                         |
| `onLanguageCreated / onLanguageDeleted / onPrimaryLanguageChanged(lang, mutations)` | `({ code, mutations })`, mutations are `{ code, primary }[]`                          |
| `onLanguageSelected(lang)`                                                          | `onLanguageSelected({ code })`                                                        |
| `onGetMutations(mutations)`                                                         | `onGetMutations({ mutations })`                                                       |

New in 2.x: `onPreviewClose()`, `onImageDelete({ items })` with items `{ name, type, path, url?, key }`, `onBlocksAndStructuresVisibilityChange({ value })`, `onTemplateUpdated()`, the nonce authentication callback `getNonce`, and the Cloud callbacks `onCloudSave`, `onCloudTemplateChange`. `onEditorError` gains the `nonceAuth` source.

Reserved keys: `onClose` and `onError` are stripped before reaching the editor. Any handler under those names is dead code in 2.x.

1.x wrapper-specific payload differences that also change: Vue/Svelte language events carried `{ lang, mutations }` → now `{ code, mutations }`; Svelte `on:onLanguageSelected` detail was a string → now `{ code }`; Svelte `on:onGetMutations` detail was an array → now `{ mutations }`; Vue/Svelte `onError` event `{ type, message, responseBody }` → `onEditorError` `{ source, message, detail }`.

## Config table (`IEmailConfig`)

| 1.x (`ITopolOptions`)                                                                        | 2.x (`IEmailConfig`)                                                                                                                                                                                                                                             |
| -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                                                                         | removed — pass the container to `render()` / the wrapper owns it                                                                                                                                                                                                 |
| `callbacks: { ... }`                                                                         | removed — callbacks sit beside `config` in the factory props                                                                                                                                                                                                     |
| `savedBlocks: ISavedBlock[] \| boolean`                                                      | `savedSections: ISavedSection[] \| boolean \| null`                                                                                                                                                                                                              |
| `premadeBlocks`, `premadeBlocksNoOverwrite`                                                  | `premadeSections: false \| sections \| { blocks, override }`                                                                                                                                                                                                     |
| `syncedRowsEnable`                                                                           | `syncedSectionsEnabled`                                                                                                                                                                                                                                          |
| `topBarOptions: string[]`                                                                    | `hideTopbarControls: string[]` (controls to hide)                                                                                                                                                                                                                |
| `defaultTemplateSettings.langs[].key`                                                        | `defaultTemplateSettings.langs[].code`                                                                                                                                                                                                                           |
| `authorize.userId: string \| number`                                                         | non-negative integer, or string matching `[A-Za-z0-9_-]+`                                                                                                                                                                                                        |
| `api.SAVE`, `api.LOAD`, `api.PREVIEW`, `api.AUTOSAVE`, `api.GET_AUTOSAVE`, `api.SYNCED_ROWS` | removed                                                                                                                                                                                                                                                          |
| `api.SAVED_BLOCKS`                                                                           | `api.SAVED_SECTIONS`                                                                                                                                                                                                                                             |
| —                                                                                            | `api.GENERATE_TEXT`, `GENERATE_PREHEADER`, `GENERATE_TRANSLATION`, `PRODUCT_CATEGORIES`, `PREMADE_TEMPLATES`, `PREMADE_TEMPLATE_CATEGORIES`, `PREMADE_TEMPLATES_KEYWORDS`                                                                                        |
| —                                                                                            | `cloud: { enabled, templateId?, features? }`                                                                                                                                                                                                                     |
| —                                                                                            | `permissions`, `enableSubjectLine`, `chatAIOnSegment`, `aiLabeledElements`, `languageMergeTag`, `productMergetagTags`, `enableFileManagerInGif`, `googlePromotionalAnnotation`, `ecm`, `lambdaStage`, countdown/loop/rating content blocks, image-editor options |

Types: `IEmailConfig` (config), `ITopolOptions` (`{ config } & IEmailCallbacks`), `IEmailEditorInstance` (returned instance), `IEmailEditorPluginOptions` (`{ stage?, loaderUrl? }`), `ISavedSection`, `IPremadeSections`, `ILanguageMutation`, `ISaveData`, `IErrorData`, `ISendTestData`, `INonceAuthCallbacks` (`getNonce`; re-exported by every wrapper), `ICloudCallbacks`, `ICloudOptions`, `ICloudFeatures`. `ISavedBlock` is a deprecated alias of `ISavedSection`. Wrapper config aliases: `IReactEmailOptions`, `IVueEmailOptions`, `ISvelteOptions` (all `IEmailConfig`).

## Loader and environments

- Default loader: `https://v4.email-assets.topol.io/loader/build.js`. The v3 loader is incompatible and rejected.
- `stage`: `"production"` (default), `"dev"`, `"staging"` (both resolve to the release loader unless the build-time override is set), or a number selecting a PR preview.
- `loaderUrl` (init option / wrapper prop) selects a local or preview loader explicitly and never falls back. The upstream local loader uses the host page origin for the iframe, so serve or proxy the editor at that origin.
- Build-time overrides for the package build: `VITE_TOPOL_V4_URL`, `VITE_TOPOL_V4_DEV_URL`, `VITE_TOPOL_V4_STAGING_URL`. `VITE_TOPOL_URL` is ignored.
- One loader URL per page. A second `init` with a different URL rejects: "Editor v4 supports one loader URL per page."
- Let `init` insert the script. A host-inserted `<script src=…loader/build.js>` or a foreign `window.TopolEmailEditor` rejects with "Cannot initialize from a host-provided loader with unknown lifecycle." An already loaded production runtime is reused.
- Numeric-stage loader failures fall back to production only before any runtime has initialized.

## Nonce authentication

Passing `getNonce` switches authentication from the public key alone to short-lived `nc_` nonces minted on the host's backend from the `sk_` secret. It is standalone: there is no separate option, the presence of the callback enables it, and templates, autosaves, images, comments, and sections keep using the host's `api.*` endpoints and callbacks. Topol Cloud is built on top of it.

```ts
const editor = await EmailEditor.init({
  config: { authorize: { apiKey: "pk_public-key", userId: "user-123" } },
  async getNonce() {
    // Runs on the host page with first-party session cookies. Your server calls Topol's
    // POST /auth/nonce with the sk_ secret and this userId. sk_ secrets never reach the browser.
    const response = await fetch("/api/topol/nonce", { method: "POST" });
    if (!response.ok) throw new Error("Could not mint a Topol nonce");
    return (await response.json()).nonce; // a fresh nc_ token
  },
});
```

Rules:

- In wrappers, `getNonce` is a React prop or a `callbacks` member (Vue/Svelte), never an event. Its type is `INonceAuthCallbacks["getNonce"]`; it moved from `ICloudCallbacks`, so only code referencing `ICloudCallbacks["getNonce"]` directly needs a change.
- The nonce endpoint is the host's responsibility: authenticate the session, derive the user, mint the nonce with the `sk_` secret server-side.
- The editor refreshes the nonce before it expires and retries once on a rejected nonce. A first nonce that cannot be obtained reports `onEditorError` with `source: "authorize"`; a later refresh failure reports `source: "nonceAuth"` while the editor keeps working on the last valid nonce.

## Topol Cloud

Cloud moves templates, autosaves, images, comments, sections, multilanguage, and test-send into Topol's backend. It runs on nonce authentication, so set up `getNonce` as above first.

```ts
const editor = await EmailEditor.init({
  config: {
    authorize: { apiKey: "pk_public-key", userId: "user-123" },
    cloud: { enabled: true, templateId: 123, features: { images: false } },
  },
  getNonce, // required; see Nonce authentication
  onCloudSave({ templateId, name, langCode }) {},
  onCloudTemplateChange({ type, templateId, name, folderId }) {}, // type: rename | delete | duplicate | move
});
```

Rules:

- `cloud.enabled` needs a `pk_` key and `getNonce`; `init` rejects without it.
- Each feature can be switched back to external behaviour with `features.<name>: false`. Cloud-active features ignore their external `api` endpoints and callbacks and warn about them. `onSave` and `onTestSend` do not fire for Cloud-owned persistence and test-send; `onSaveAndClose`, lifecycle, error, and notification callbacks still do. `instance.save()` works in either mode.
- `cloud.templateId` opens a stored template; omitting it opens the Cloud template browser. `config.templateId` is ignored in Cloud mode. A new template's id arrives via `onCloudSave`.

## Leftover grep (must return nothing after migration)

```sh
grep -rnE "window\.TopolPlugin|TopolPlugin\.(init|save|load|destroy|set|toggle|refresh|update|create|delete|select|get|open|change|choose|undo|redo)|\bid: *['\"]#|callbacks: *\{|savedBlocks|setSavedBlocks|premadeBlocks|syncedRowsEnable|refreshSyncedRows|topBarOptions|onBlockSave|onBlockRemove|onBlockEdit|SAVED_BLOCKS|SYNCED_ROWS|VITE_TOPOL_URL\b|\bonError\(|\bonClose\(" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git \
  --exclude=package-lock.json --exclude=yarn.lock --exclude=pnpm-lock.yaml
```

`onError` / `onClose` hits inside `LandingPageEditor` usage are fine; that API is unchanged.
