# Shared configuration, callbacks and methods

Applies to every package. The framework references cover the wiring; this file
covers what goes *inside* the options object and what the instance can do.

## Required config

Both editors require exactly one thing:

```ts
authorize: {
  apiKey: string,             // public — ships to the browser
  entityId: string | number,  // the account the editor works for (your user or tenant id)
}
```

`entityId` decides where that account's images are stored and counts as one
billed user. For the Email Editor it must be a non-negative integer or a
string matching `[A-Za-z0-9_-]+`; anything else makes `init()` reject. Map emails
or other ids to a URL-safe value (for example a hash). `userId` is a deprecated
alias of `entityId`; do not use it in new code. The type is `IAuthorize`.

Neither editor takes a container id in its config. The core `init()` resolves to
an instance and you call `instance.render(selectorOrElement)`; the framework
wrappers own their container and call `render` for you.

The core factory props are `{ config, ...callbacks }` for **both** editors:
settings live under `config`, callbacks sit next to it at the top level.

## Commonly used optional config (Email Editor, `IEmailConfig`)

| Option | Type | Notes |
| --- | --- | --- |
| `api` | object | Endpoints on your own backend (`AUTOSAVES`, `IMAGE_UPLOAD`, `FOLDERS`, `SAVED_SECTIONS`, `CONVERSATIONS`, `COMMENTS`, `PRODUCTS`, `FEEDS`, `PREMADE_TEMPLATES`, …) |
| `apiAuthorizationHeader` | `string \| object` | Header sent to your `api` endpoints — this is how you actually secure storage |
| `templateId` | `string \| number` | Ties comments/autosaves to one template |
| `light`, `theme` | boolean / object | UI theming |
| `language` | string | Editor UI language |
| `role` | `'manager' \| 'editor' \| 'reader'` | What the current user may do |
| `customFileManager` | boolean | Suppress the built-in file manager, handle `onOpenFileManager` yourself |
| `customBlocks`, `savedSections`, `premadeSections` | arrays / objects | Content library ("sections" were called "blocks" in the v3 API) |
| `syncedSectionsEnabled` | boolean | Synced sections |
| `enableAutosaves`, `autosaveInterval` | boolean / number | Autosave |
| `enableComments` | boolean | Collaboration comments |
| `multilingual`, `defaultTemplateSettings.langs` (`{ code, primary }[]`) | — | Multilanguage templates |
| `mergeTags`, `productMergeTags` | arrays | Merge tags |
| `removeTopBar`, `hideTopbarControls`, `windowBar` | boolean / string[] | Build your own top bar (`hideTopbarControls` lists controls to hide) |
| `saveAndCloseButton` | `'dropdown' \| 'separate'` | Layout of Save and Save & Close |
| `hideSelectionPanel`, `hideSelectionPanelExpand` | boolean | Right-side selection panel |
| `disableShortcuts` | `boolean \| EmailEditorShortcut[]` | Turn keyboard shortcuts off (e.g. `['save']` so Ctrl/Cmd+S cannot start a save) |
| `chatAI`, `disableAiAssistant` | boolean | AI features |
| `cloud` | `{ enabled, templateId?, features? }` | Topol Cloud; requires `getNonce` |
| `countdown` | object | Host settings of the Countdown block (a paid add-on enabled per account) |

Option reference: <https://docs.topol.io/email-editor/reference/topol-options.html>.
Use the exported `IEmailConfig` type as the source of truth for names; do not
guess. A misspelled Email Editor option is ignored without an error.

The Landing Page Editor's config type is `ILandingPageConfig`. It is validated at
runtime, at init and on every `updateOptions()`: a value of the wrong type stops
the editor from starting and is reported through `onEditorError` with
`source: 'init'`. Unknown keys are ignored. Reference:
<https://docs.topol.io/landing-page-editor/reference/topol-options.html>.

## Callbacks

Every callback of **both** editors takes no arguments or a single object, in
every package. Wrappers deliver the same object (React and Svelte props, Vue
events and `callbacks` members).

Email Editor (`IEmailCallbacks`):

```ts
onSave({ json, html, mutations, syncedSections })          // may return a Promise; the editor awaits it
onSaveAndClose({ json, html, mutations, syncedSections })  // same
onTestSend({ email, json, html })                          // email: string | string[]
onEditorError({ source, message, detail })
onEditorClose()   // user closed the editor; may return a Promise
onInit()          // editor authenticated
onLoaded()        // a template was loaded
onSavedSectionSave({ section })                            // section: { definition, translation? }; you assign id and name
onSavedSectionRemove({ id }) / onSavedSectionEdit({ id })
onUndoChange({ count }) / onRedoChange({ count })
onPreview({ html }) / onPreviewClose()
onAlert({ notification })
onUnsavedChangesChange({ hasUnsavedChanges })
onHiddenElementsVisibilityChange({ visible })
onOpenFileManager() / onImageDelete({ items })
onOpenCustomBlockDialog({ block })
onTemplateRename({ name }) / onTemplateUpdated()
onTestingEmailsChange({ emails })
onBannerClick({ json, html })
onLanguageCreated({ code, mutations }) / onLanguageDeleted({ code, mutations })
onPrimaryLanguageChanged({ code, mutations }) / onLanguageSelected({ code })
onGetMutations({ mutations })                              // mutations: { code, primary }[]
getNonce(): Promise<string>                                // optional; enables nonce authentication
onCloudSave({ templateId, name?, langCode? }) / onCloudTemplateChange({ type, templateId, name?, folderId? })
```

Landing Page Editor (`ILandingPageCallbacks`): `onSave({ json, html, syncedSections? })`,
`onSaveAndClose(...)` (same payload), `onEditorError({ source, message, detail })`,
`onEditorClose()`, `onInit()`, `onLoaded()`, `onOpenFileManager()`,
`onImageDelete({ items })`, `onUndoChange({ count })`, `onRedoChange({ count })`,
`onPreview({ html })`, `onPreviewClose()`, `onAlert({ notification })`,
`onUnsavedChangesChange({ hasUnsavedChanges })`, `onOpenCustomBlockDialog({ block })`,
`onTemplateRename({ name })`, `getNonce()`. Without `onSaveAndClose` the payload
goes to `onSave` and the editor closes itself.

`onEditorError`'s `source` is the exported `EditorErrorSource`: `integration`
(the wrapper or loader could not start the editor), `init`, `authorize`,
`nonceAuth`, `network`, `load`, `updateTemplate`, `save`, `testSend`,
`cloudTemplates`, `cloudSync`.

`onClose` and `onError` are reserved by the iframe bridge and never reach either
editor — always use `onEditorClose` / `onEditorError`.

`onLoaded` fires after every `load()`. Load the first template from `onInit` (or
after `render()`), never from `onLoaded`, or it loops.

Deprecated, still fired but logged as a warning — do not use in new code:
`onEdittedWithoutSaveChanged({ value })` → `onUnsavedChangesChange`,
`updateTestingEmailAddresses({ emails })` → `onTestingEmailsChange`,
`onBlocksAndStructuresVisibilityChange({ value })` → `onHiddenElementsVisibilityChange`.

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
await editor.updateApiAuthorizationHeader(header);   // object is merged; a null value removes that header
await editor.chooseFile(url);                        // answer to onOpenFileManager
await editor.updateCustomBlockContent(content); await editor.refreshComments(key);
await editor.setActiveMembers(names);
await editor.createLanguage(code); await editor.deleteLanguage(code);
await editor.selectLanguage(code); await editor.setPrimaryLanguage(code);
await editor.translateLanguage(code, sourceCode?); await editor.translateAllLanguages();
const mutations = await editor.getMutations();   // { code, primary }[]
await editor.toggleSelectionPanel(); await editor.toggleHiddenElementsVisibility();
await editor.toggleChatAI(); await editor.toggleAutosaves(); await editor.toggleComments();
await editor.openPremadeTemplatesSelection();
await editor.destroy();             // close() is an alias
```

Only one save runs at a time: `save()` rejects with an error whose `code` is
`"SAVE_IN_PROGRESS"` while another save is running. Check `err.code`, not the
class. Deprecated aliases: `toggleControlPanel()` → `toggleSelectionPanel()`,
`toggleBlocksAndStructuresVisibility()` → `toggleHiddenElementsVisibility()`.

The wrappers expose the instance: React `ref.current?.getEditor()` or the
`onReady(editor)` prop; Vue template ref `getEditor()` or `@ready`; Svelte
`onReady(editor)` or `bind:this` + `getEditor()`. Wait for `onInit` before calling
methods that need an authenticated editor.

## Instance methods (Landing Page Editor)

`LandingPageEditor.init()` resolves to an `ILandingPageEditorInstance`. Every
method returns a promise.

```ts
await instance.render(selectorOrElement);   // nothing appears until you do
await instance.load(template);              // JSON string or parsed object
await instance.save();                      // resolves after onSave settles; SAVE_IN_PROGRESS as above
await instance.updateOptions(partialConfig);            // authorize excluded; rejects on invalid options
await instance.togglePreview(); await instance.setPreviewSize(index);   // 0 mobile … 3 desktop
await instance.undo(); await instance.redo();
await instance.chooseFile(url); await instance.setTemplateName(name);
await instance.refreshSyncedSections(); await instance.refreshComments(key);
await instance.openPremadeTemplatesSelection(); await instance.toggleSelectionPanel();
await instance.toggleAutosaves(); await instance.toggleComments(); await instance.setActiveMembers(names);
await instance.updateCustomBlockContent(content);
await instance.updateApiAuthorizationHeader(header); await instance.createNotification(notification);
await instance.destroy();                   // close() is an alias
```

In React/Vue the component calls `render` for you. Its ref exposes `load()`,
`save()` and `getEditor()` (the instance, for everything else), and
`onReady` / `@ready` hands over the instance once the iframe has rendered.

## Stage / loader

The Email Editor loads `https://v4.email-assets.topol.io/loader/build.js` at init;
the Landing Page Editor loads `https://v1.page-assets.topol.io/loader/build.js`.
The core `init()` takes a second argument `{ stage?, loaderUrl? }`; the wrappers
take the same as `stage` / `loaderUrl` props. Leave both unset in production code.

- Email Editor: `dev` / `staging` / a numeric stage point at Topol's internal and
  preview builds, not at your environments. Let `init()` insert the script — a
  host-inserted loader `<script>` is rejected.
- Landing Page Editor: `stage` is accepted and ignored (production only).
- Both: `loaderUrl` selects an explicit loader (a preview or local build) and
  never falls back. One loader URL per editor per page; switching needs a reload.

## Nonce authentication and Topol Cloud

Optional, and the most secure way to run either editor. Passing a `getNonce`
callback switches the editor from the public API key alone to short-lived `nc_`
nonces created on the app's backend with the Secret key. There is no separate
option: the presence of the callback enables it, and the `api.*` endpoints and
callbacks keep working as before.

```ts
async getNonce() {
  // Runs on the host page with first-party session cookies.
  const response = await fetch('/api/topol/nonce', { method: 'POST' });
  if (!response.ok) throw new Error('Could not get a Topol nonce');
  return (await response.json()).nonce;   // a fresh nc_ token
}
```

- Where it goes: core — next to the other callbacks; React and Svelte — the
  `getNonce` prop; Vue — a member of the `callbacks` prop (never an event). It
  works on `EmailEditor` and `LandingPageEditor`. Type: `INonceAuthCallbacks['getNonce']`.
- The backend endpoint is the app's job: answer only signed-in users, bind the
  nonce to the same `entityId` as `authorize.entityId`, expire it within an hour,
  and keep the Secret key (`sk_`) on the server. The signing code and the
  alternative Topol minting endpoint are in the guides:
  <https://docs.topol.io/email-editor/guide/nonce-authentication.html>,
  <https://docs.topol.io/landing-page-editor/guide/nonce-authentication.html>.
- A first nonce that cannot be obtained reports `onEditorError` with
  `source: 'authorize'`; a failed refresh reports `source: 'nonceAuth'` while the
  editor keeps working on the last valid nonce.
- `cloud: { enabled: true }` (Email Editor) moves storage into Topol Cloud and
  requires `getNonce`; `init` rejects without it.

Only add these when the user asks; scaffold the backend endpoint as a clearly
marked TODO rather than inventing the app's session handling.

## TypeScript

Every package re-exports the type surface; import types from the package you
installed. Data going into or out of an editor is `IEmail*` / `ILandingPage*`;
types of the editor object are `IEmailEditor*` / `ILandingPageEditor*`.

```ts
// core
import type { IEmailOptions, IEmailConfig, IEmailCallbacks, IEmailEditorInstance,
              IEmailEditorPluginOptions, IEmailSavePayload, IEmailErrorPayload,
              IEmailTestSendPayload, ISavedSection, ISavedSectionDraft, ILanguageMutation,
              ILandingPageOptions, ILandingPageConfig, ILandingPageCallbacks,
              ILandingPageEditorInstance, ILandingPageEditorPluginOptions,
              ILandingPageSavePayload, ILandingPageErrorPayload,
              IAuthorize, EditorErrorSource, INonceAuthCallbacks, ICloudOptions } from '@topol.io/editor';

// React
import type { IReactEmailOptions, IEmailEditorRef, IEmailEditorProps,
              IReactLandingPageOptions, ILandingPageEditorRef, ILandingPageEditorProps } from '@topol.io/editor-react';

// Vue
import type { IVueEmailOptions, IVueLandingPageOptions } from '@topol.io/editor-vue';

// Svelte
import type { ISvelteEmailOptions, IEmailEditorProps } from '@topol.io/editor-svelte';
```

`IEmailOptions` / `ILandingPageOptions` are the core factory props
`{ config, ...callbacks }`. The config types are `IEmailConfig` (wrapper aliases
`IReactEmailOptions`, `IVueEmailOptions`, `ISvelteEmailOptions`) and
`ILandingPageConfig` (`IReactLandingPageOptions`, `IVueLandingPageOptions`).
Template JSON is typed `unknown`; there is no exported template type.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Blank page, no iframe | Container has no height, or `render()` was never called (core) |
| `EmailEditor.init requires a browser` | Init ran during SSR — mount client-side only (see `react.md`) |
| `Editor v4 requires { config: { authorize, ... } }` | v3-style options (`id`, nested `callbacks`) or an old example — use `{ config, ...callbacks }` |
| `authorize.entityId must be a URL-safe string…` | `entityId` contains `@`, `.`, spaces, … — map it to a URL-safe value |
| Handler never fires | Named `onError` / `onClose` / `onBlock*` — use the current names |
| `has no exported member` `TopolPlugin` / `TopolEditor` / `ITopolOptions` / `ISaveData` / `ISvelteOptions`, or no default export | Names from 0.x or an earlier alpha — use the ones in the TypeScript section |
| Svelte: `on:onSave` / `callbacks` do nothing, or the package fails to load | The Svelte wrapper needs Svelte 5 and takes callbacks as props (see `svelte.md`) |
| Editor loads then errors on save | `api` endpoints/auth header, not the editor — check `onEditorError`'s `detail` |
| `save()` rejects with `code: "SAVE_IN_PROGRESS"` | A save is already running (e.g. Ctrl/Cmd+S plus your own button) — wait for it or set `disableShortcuts` |
| `Cannot initialize from a host-provided loader` | A `<script src=…loader/build.js>` was added by hand — remove it |
| `supports one loader URL per page` | Two different `stage` / `loaderUrl` values on one page — use one, or reload |
| Installed the old API by accident | `latest` is 0.3.0; pin the 1.x prerelease, `1.0.0-alpha.11`+ (editor-react `.12`+) |
| Two editors / duplicate iframes | Two mounts of the component, or a manual `init()` alongside a wrapper |
