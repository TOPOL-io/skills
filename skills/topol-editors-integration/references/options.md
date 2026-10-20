# Shared configuration, callbacks and methods

Applies to every package. The framework references cover the wiring; this file
covers what goes *inside* the options object and what the instance can do.

## Required config

Both editors require exactly one thing:

```ts
authorize: {
  apiKey: string,           // public — ships to the browser
  userId: string | number,  // your app's user id, for saved sections / comments scoping
}
```

For the Email Editor, `userId` must be a non-negative integer or a string
matching `[A-Za-z0-9_-]+`; anything else makes `init()` reject. Map emails or
other ids to a URL-safe value (for example a hash).

Neither editor takes a container id in its config. The core `init()` resolves to
an instance and you call `instance.render(selectorOrElement)`; the framework
wrappers own their container and call `render` for you.

The core factory props are `{ config, ...callbacks }` for **both** editors:
settings live under `config`, callbacks sit next to it at the top level.

## Commonly used optional config (Email Editor, `IEmailConfig`)

| Option | Type | Notes |
| --- | --- | --- |
| `api` | object | Endpoints on your own backend (`AUTOSAVES`, `IMAGE_UPLOAD`, `FOLDERS`, `SAVED_SECTIONS`, `TEST_EMAIL`, …) |
| `apiAuthorizationHeader` | `string \| object` | Header sent to your `api` endpoints — this is how you actually secure storage |
| `templateId` | `string \| number` | Ties comments/autosaves to one template |
| `light`, `theme` | boolean / object | UI theming |
| `language` | string | Editor UI language |
| `customFileManager` | boolean | Suppress the built-in file manager, handle `onOpenFileManager` yourself |
| `customBlocks`, `savedSections`, `premadeSections` | arrays / objects | Content library ("sections" were called "blocks" in the v3 API) |
| `syncedSectionsEnabled` | boolean | Synced sections |
| `enableAutosaves`, `autosaveInterval` | boolean / number | Autosave |
| `enableComments` | boolean | Collaboration comments |
| `multilingual`, `defaultTemplateSettings.langs` (`{ code, primary }[]`) | — | Multilanguage templates |
| `removeTopBar`, `hideTopbarControls`, `windowBar` | boolean / string[] | Build your own top bar (`hideTopbarControls` lists controls to hide) |
| `chatAI`, `disableAiAssistant` | boolean | AI features |
| `cloud` | `{ enabled, templateId?, features? }` | Topol Cloud; requires `getNonce` |
| `enableCountdown`, `countdown` | boolean / object | Countdown block (paid add-on) |

Option reference: <https://docs.topol.io/email-editor/reference/topol-options.html>.
Use the exported `IEmailConfig` type as the source of truth for names; do not
guess.

The Landing Page Editor's config (`ILandingPageOptions['config']`) is validated
at runtime — an unknown or wrongly-typed option is rejected rather than ignored.
`role: 'manager' | 'editor' | 'reader'` is Landing Page Editor only.

## Callbacks

Every callback of **both** editors takes no arguments or a single object, in
every package. Wrappers deliver the same object (Vue/Svelte events: the event
argument / `event.detail`).

Email Editor (`IEmailCallbacks`):

```ts
onSave({ json, html, mutations, syncedSections })          // may return a Promise; the editor awaits it
onSaveAndClose({ json, html, mutations, syncedSections })  // same
onTestSend({ email, json, html })                          // email: string | string[]
onEditorError({ source, message, detail })
onEditorClose()
onInit()          // editor authenticated
onLoaded()        // template loaded
onSavedSectionSave({ section }) / onSavedSectionRemove({ id }) / onSavedSectionEdit({ id })
onUndoChange({ count }) / onRedoChange({ count })
onPreview({ html }) / onPreviewClose()
onAlert({ notification })
onEdittedWithoutSaveChanged({ value })                     // upstream spelling
onOpenFileManager() / onImageDelete({ items })
onOpenCustomBlockDialog({ block })
onTemplateRename({ name }) / onTemplateUpdated()
updateTestingEmailAddresses({ emails })
onBannerClick({ json, html })
onBlocksAndStructuresVisibilityChange({ value })
onLanguageCreated({ code, mutations }) / onLanguageDeleted({ code, mutations })
onPrimaryLanguageChanged({ code, mutations }) / onLanguageSelected({ code })
onGetMutations({ mutations })                              // mutations: { code, primary }[]
getNonce(): Promise<string>                                // optional; enables nonce authentication
onCloudSave({ templateId, name?, langCode? }) / onCloudTemplateChange({ type, templateId, name?, folderId? })
```

Landing Page Editor (`ILandingPageCallbacks`): `onSave({ json, html })`,
`onSaveAndClose({ json, html })`, `onEditorError({ source, message, detail })`,
`onEditorClose()`, `onInit()`, `onLoaded()`, `onOpenFileManager()`,
`onImageDelete({ items })`, `onUndoChange({ count })`, `onRedoChange({ count })`,
`onPreview({ html })`, `onPreviewClose()`, `onAlert({ notification })`,
`onEdittedWithoutSaveChanged({ value })`, `onOpenCustomBlockDialog({ block })`,
`onTemplateRename({ name })`.

`onClose` and `onError` are reserved by the iframe bridge and never reach either
editor — always use `onEditorClose` / `onEditorError`.

## Instance methods (Email Editor)

`EmailEditor.init()` resolves to an `IEmailEditorInstance`. Every method returns
a promise. Each `init()` creates an independent instance; there is no global
singleton.

```ts
await editor.render(container);     // selector or HTMLElement; must exist and have a height
await editor.load(json);
await editor.save({ langCode? });   // resolves after onSave settles
await editor.updateTemplate(json, { skipSnapshot });
await editor.updateOptions(partialConfig);   // authorize excluded
await editor.togglePreview(); await editor.togglePreviewSize(); await editor.toggleDarkMode();
await editor.undo(); await editor.redo();
await editor.changeEmailToMobile(); await editor.changeEmailToDesktop();
await editor.setSavedSections(sections); await editor.refreshSyncedSections();
await editor.setMergeTags(tags); await editor.setTemplateName(name);
await editor.createNotification(notification);
await editor.updateApiAuthorizationHeader(header);
await editor.createLanguage(code); await editor.deleteLanguage(code);
await editor.selectLanguage(code); await editor.setPrimaryLanguage(code);
await editor.translateLanguage(code, sourceCode?); await editor.translateAllLanguages();
const mutations = await editor.getMutations();   // { code, primary }[]
await editor.toggleControlPanel(); await editor.openPremadeTemplatesSelection();
await editor.destroy();
```

The wrappers expose the instance: React `ref.current?.getEditor()` or the
`onReady(editor)` prop; Vue template ref `getEditor()` or `@ready`; Svelte
`bind:this` + `getEditor()` or `on:ready`. Wait for `onInit` before calling
methods that need an authenticated editor.

## Instance methods (Landing Page Editor)

```ts
await instance.render(selectorOrElement)   // nothing appears until you do
instance.load(template)
instance.save()                            // triggers onSave
await instance.destroy()
```

In React/Vue, the wrapper calls `render` for you and exposes `load` / `save`
through a ref.

## Stage / loader

The Email Editor loads `https://v4.email-assets.topol.io/loader/build.js` at init.
The core `init()` takes a second argument `{ stage?, loaderUrl? }`; the wrappers
take the same as `stage` / `loaderUrl` props. Leave both unset in production code.
`dev` / `staging` / a numeric stage point at Topol's internal and preview builds,
not at your environments; `loaderUrl` selects an explicit v4 loader and never
falls back. Only one loader URL works per page; let `init()` insert the script
(a host-inserted loader `<script>` is rejected).

The Landing Page Editor ignores `stage` (production loader only).

## Nonce authentication and Topol Cloud

Optional. Passing a `getNonce` callback that fetches a fresh `nc_` token from the
app's authenticated backend switches the Email Editor to nonce authentication;
`cloud: { enabled: true }` moves storage into Topol Cloud and requires
`getNonce`. Both need a `pk_` key, and the `sk_` secret stays on the server. Only
add them when the user asks; scaffold the backend endpoint as a clearly marked
TODO.

## TypeScript

Every package re-exports the type surface; import types from the package you
installed:

```ts
// core
import type { IEmailConfig, ITopolOptions, IEmailCallbacks, IEmailEditorInstance,
              IEmailEditorPluginOptions, ISaveData, IErrorData, ISendTestData,
              ISavedSection, ILanguageMutation, INonceAuthCallbacks, ICloudOptions,
              ILandingPageOptions, ILandingPageCallbacks, ILandingPageEditorInstance,
              TopolSection } from '@topol.io/editor';

// React
import type { IReactEmailOptions, IEmailEditorRef, IEmailEditorProps,
              IReactLandingPageOptions, LandingPageEditorProps, LandingPageEditorRef } from '@topol.io/editor-react';

// Vue
import type { IVueEmailOptions, IVueLandingPageOptions } from '@topol.io/editor-vue';

// Svelte
import type { ISvelteOptions, ISvelteEmailEvents } from '@topol.io/editor-svelte';
```

`ITopolOptions` means the core factory props `{ config, ...callbacks }`; the
config type is `IEmailConfig` (wrapper aliases `IReactEmailOptions`,
`IVueEmailOptions`, `ISvelteOptions`).

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Blank page, no iframe | Container has no height, or `render()` was never called (core) |
| `EmailEditor.init requires a browser` | Init ran during SSR — mount client-side only (see `react.md`) |
| `Editor v4 requires { config: { authorize, ... } }` | v3-style options (`id`, nested `callbacks`) or an old example — use `{ config, ...callbacks }` |
| `authorize.userId must be a URL-safe string…` | `userId` contains `@`, `.`, spaces, … — map it to a URL-safe value |
| Handler never fires | Named `onError` / `onClose` / `onBlock*` — use the v4 names |
| Editor loads then errors on save | `api` endpoints/auth header, not the editor — check `onEditorError`'s `detail` |
| `Cannot initialize from a host-provided loader` | A `<script src=…loader/build.js>` was added by hand — remove it |
| Installed the old API by accident | `latest` is 0.3.0 (v3 API); pin the `alpha` version, `1.0.0-alpha.10`+ (editor-react `.11`+) |
| Two editors / duplicate iframes | Two mounts of the component, or a manual `init()` alongside a wrapper |
