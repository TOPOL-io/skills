# Full changed surface, 0.x → 1.x

Reference list. The per-framework files cover the common cases; use this for
anything they miss.

## Still works (deprecated aliases, warn but do not break)

| 0.x symbol | Status in 1.x |
| --- | --- |
| `import TopolPlugin from '@topol.io/editor'` | Works — still the default export |
| `TopolPlugin` (named, core) | Works — alias of `EmailEditor` |
| `TopolEditor` (React/Vue **named** import) | Works — wraps `EmailEditor`, logs a console deprecation warning on mount |
| `TopolPlugin.init(options, { stage })` | Unchanged signature |

## Hard breaks

| What | 0.x | 1.x | Affects |
| --- | --- | --- | --- |
| React default export | `import TopolEditor from '@topol.io/editor-react'` | no default export — `import { EmailEditor } from '@topol.io/editor-react'` | React |
| React container id | `#editor` | `#topol-email-editor-id` | React CSS/DOM |
| Vue container id | `#topol-editor-id` | `#topol-email-editor-id` | Vue CSS/DOM |
| `ITopolOptions` from a wrapper package | alias of the wrapper's own options (`Omit<ITopolOptions,'id'\|'callbacks'>`) | re-exports the **core** `ITopolOptions`, which still has `id` and `callbacks` | React, Vue |
| `ISaveData` / `ISendTestData` / `IErrorData` import path (Vue) | `from './TopolEditor.vue'` (or the package root) | from the package root, sourced from core | Vue |

The `ITopolOptions` change is the sneaky one: code that annotated an options
object with `ITopolOptions` from a wrapper package now fails because `id` and
`callbacks` are required. Use the wrapper-specific names instead:

- React email: `IReactEmailOptions` (`IReactTopolOptions` kept as a deprecated alias)
- React landing page: `IReactLandingPageOptions`
- Vue email: `IVueEmailOptions` (`IVueOptions` still exported)
- Vue landing page: `IVueLandingPageOptions`

## Callback signature changes (core + React + Vue)

Additive parameters — existing handlers keep compiling, but the payloads carry
more now:

- `onSave(json, html)` → `onSave(json, html, mutations, syncedSections)`
- `onSaveAndClose(json, html)` → `onSaveAndClose(json, html, mutations, syncedSections)`
- `onError(type, message)` → `onError(type, message, responseBody?)`
- `onTestSend(email, json, html)` — **`email` is now `string | string[]`** (this
  one *can* break a typed handler, and can break at runtime if the value is fed
  straight to an API expecting a single address)

`mutations` is `{ key: string; primary: boolean }[]`; `syncedSections` is `number[]`.

New multilingual callbacks/events (opt-in, ignore if unused):
`onLanguageCreated`, `onLanguageDeleted`, `onLanguageSelected`,
`onPrimaryLanguageChanged`, `onGetMutations`.

In Vue these arrive as emitted events with an object payload, e.g.
`@on-language-created="({ lang, mutations }) => …"`.

## New exports in 1.x

Core (`@topol.io/editor`): `EmailEditor`, `LandingPageEditor`, and the types
`IEmailCallbacks`, `ILandingPageOptions`, `ILandingPageCallbacks`,
`ILandingPageEditorInstance`, `ISaveData`, `IErrorData`, `ISendTestData`,
`TopolSection`, plus the previously available `INotification`, `ISavedBlock`,
`IMergeTag`, `IMergeTagGroup`, `IContentBlockOptions`, `ITheme`,
`IAuthHeaderConfig`, `IFont`, `IAPI`, `ICustomBlockData`, `IStage`.

React/Vue also re-export the core editors under aliases for programmatic use:
`CoreEmailEditor`, `CoreLandingPageEditor`.

## Package-by-package summary

- `@topol.io/editor` — no breaking runtime change; new named exports.
- `@topol.io/editor-react` — default export removed, container id changed.
- `@topol.io/editor-vue` — container id changed, `ITopolOptions` meaning changed.
- `@topol.io/editor-svelte` — **unchanged public API**; version bump only. No
  Svelte landing-page component exists yet; a Svelte app that needs it must call
  the core `LandingPageEditor` directly (see `landing-page-editor.md`).
