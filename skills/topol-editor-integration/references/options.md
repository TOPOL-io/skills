# Shared configuration, callbacks and methods

Applies to every package. The framework references cover the wiring; this file
covers what goes *inside* the options object.

## Required config

Both editors require exactly two things:

```ts
authorize: {
  apiKey: string,           // public — ships to the browser
  userId: string | number,  // your app's user id, for saved blocks / comments scoping
}
```

The core Email Editor additionally requires `id` — a CSS selector for the
container (`'#editor'`). The framework wrappers own their container and inject
`id` for you, which is why their options types are `Omit<ITopolOptions, 'id' | 'callbacks'>`.

The Landing Page Editor takes no `id` in its options; you pass the selector to
`instance.render(selector)` instead.

## Commonly used optional config

Same names on both editors unless noted:

| Option | Type | Notes |
| --- | --- | --- |
| `api` | object | Endpoints for template/image storage on your own backend |
| `apiAuthorizationHeader` | `string \| object` | Header sent to your `api` endpoints — this is how you actually secure storage |
| `light` | boolean | Light UI theme |
| `theme` | object | Full theming |
| `language` | object | Editor UI language |
| `customFileManager` | boolean | Suppress the built-in file manager, handle `onOpenFileManager` yourself |
| `customBlocks`, `savedBlocks`, `premadeBlocks` | arrays | Content library |
| `templateId` | `string \| number` | Ties comments/autosaves to one template |
| `enableAutosaves`, `autosaveInterval` | boolean / number | Autosave |
| `enableComments` | boolean | Collaboration comments |
| `removeTopBar`, `hideTopbarControls`, `windowBar` | boolean / string[] | Build your own top bar |
| `chatAI`, `disableAiAssistant` | boolean | AI features |
| `role` | `'manager' \| 'editor' \| 'reader'` | LPE only |
| `apiBlocks`, `customLanguagePreset`, `betaFeatures` | — | Email editor only |

Full list: <https://docs.topol.io/reference/topol-options.html>. The Landing Page
Editor's config is validated by Zod at runtime — an unknown or wrongly-typed
option is rejected rather than ignored, so do not guess option names for LPE.

## Callbacks

Both editors expose the same core set. The **shapes differ per package**:

- core (`@topol.io/editor`): positional arguments, inside `callbacks: { … }` for
  the Email Editor, and at the **top level** next to `config` for the LPE.
- React: positional arguments as props (`onSave={(json, html, mutations, syncedSections) => …}`).
- Vue and Svelte: a **single object** per event (`{ json, html, mutations, syncedSections }`).

Email Editor callbacks: `onSave`, `onSaveAndClose`, `onTestSend`, `onLoaded`,
`onInit`, `onClose`, `onPreview`, `onAlert`, `onError`, `onOpenFileManager`,
`onBlockSave`, `onBlockRemove`, `onBlockEdit`, `onUndoChange`, `onRedoChange`,
`onBannerClick`, `onEdittedWithoutSaveChanged`, `onOpenCustomBlockDialog`,
`onTemplateRename`, `updateTestingEmailAddresses`, plus the multilanguage set
`onLanguageCreated`, `onLanguageDeleted`, `onLanguageSelected`,
`onPrimaryLanguageChanged`, `onGetMutations`.

Landing Page Editor callbacks: the same list **minus** `onTestSend`,
`updateTestingEmailAddresses` and the multilanguage set.

Signatures worth knowing before you type a handler:

```ts
onSave(json, html, mutations, syncedSections)  // email editor — 4 args
onSave(json, html)                             // landing page editor — 2 args
onTestSend(email: string | string[], json, html)   // email only; email may be an array
onError(type: string, message: string, responseBody?: unknown)
```

## Imperative methods (Email Editor)

Import the editor object and call it anywhere — it drives the singleton editor
instance, so this works from a custom top bar in any framework:

```ts
import { EmailEditor } from '@topol.io/editor';

EmailEditor.save(lang?);          // triggers onSave
EmailEditor.load(json);           // load a template
EmailEditor.updateTemplate(json, { skipSnapshot });
EmailEditor.updateOptions(partialOptions);
EmailEditor.destroy();
EmailEditor.togglePreview(); EmailEditor.togglePreviewSize(); EmailEditor.toggleDarkMode();
EmailEditor.undo(); EmailEditor.redo();
EmailEditor.changeEmailToMobile(); EmailEditor.changeEmailToDesktop();
EmailEditor.setSavedBlocks(blocks); EmailEditor.setMergeTags(tags);
EmailEditor.setTemplateName(name); EmailEditor.createNotification(notification);
EmailEditor.updateApiAuthorizationHeader(header);
EmailEditor.createLanguage(lang); EmailEditor.deleteLanguage(lang);
EmailEditor.selectLanguage(lang); EmailEditor.setPrimaryLanguage(lang);
EmailEditor.getMutations();
EmailEditor.toggleControlPanel(); EmailEditor.openPremadeTemplatesSelection();
```

The wrappers re-export it as `CoreEmailEditor` (React/Vue) so you don't need a
second import of the core package.

## Imperative methods (Landing Page Editor)

The LPE is **instance-based**, not a singleton. `init()` resolves with:

```ts
instance.render(selector)   // must be called once — nothing appears until you do
instance.load(template)
instance.save()             // triggers onSave
instance.destroy()
```

In React/Vue, the wrapper calls `render` for you and exposes `load` / `save`
through a ref.

## Stage / loader

Both editors load a script from Topol's CDN at init. The wrappers and the core
`init()` take a second argument: `{ stage: 'production' | 'dev' | 'staging' }`,
default `'production'`. Leave it at the default unless Topol told you otherwise —
`dev`/`staging` point at Topol's internal builds, not at your environments. The
LPE ignores `stage` entirely (production loader only).

## TypeScript

Every package re-exports the type surface, so import types from the package you
installed rather than reaching into the core:

```ts
// core
import type { ITopolOptions, ILandingPageOptions, ILandingPageEditorInstance,
              IEmailCallbacks, ILandingPageCallbacks, ISaveData, IErrorData,
              ISendTestData, INotification, ISavedBlock, TopolSection } from '@topol.io/editor';

// React
import type { IReactEmailOptions, IReactLandingPageOptions,
              EmailEditorProps, LandingPageEditorProps, LandingPageEditorRef } from '@topol.io/editor-react';

// Vue
import type { IVueEmailOptions, IVueLandingPageOptions } from '@topol.io/editor-vue';

// Svelte
import type { ITopolOptions } from '@topol.io/editor-svelte';  // = ISvelteOptions
```

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Blank page, no iframe | Container has no height, or you never called `instance.render()` (LPE) |
| `window.TopolPlugin is undefined` | Init ran during SSR — the loader needs a browser (see `react.md`) |
| Editor loads then errors on save | `api` endpoints/auth header, not the editor — check `onError`'s `responseBody` |
| Installed 0.3.0 by accident | `latest` is still 0.x; install the resolved `alpha` version explicitly |
| Two editors / duplicate iframes | Two mounts of the component, or a manual `init()` alongside a wrapper |
