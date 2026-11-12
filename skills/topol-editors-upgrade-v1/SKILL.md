---
name: topol-editors-upgrade-v1
description: 'Upgrade a host app from `@topol.io/editor*` 0.x or any earlier 1.x alpha to the current 1.x (`1.0.0-alpha.11`+, Email Editor v4 API). Covers `@topol.io/editor`, `-react`, `-vue`, `-svelte` and the Landing Page Editor wrappers. Use when asked to upgrade, migrate, or bump Topol editor packages to 1.x / v1 or Email Editor v4, to port `TopolPlugin` / `TopolEditor` code to the `EmailEditor` instance API, when an install throws "Editor v4 requires { config: ... }", callbacks stop firing, Svelte `on:` events stop working, or "no default export" / "has no exported member" errors for `TopolPlugin`, `TopolEditor`, `ITopolOptions`, `ISaveData` or `ISvelteOptions` appear after a Topol bump, to rename `authorize.userId` to `entityId`, or to add nonce authentication (`getNonce`) or Topol Cloud (`cloud`) to an existing integration. For a first-time integration use topol-editors-integration.'
---

# Upgrade `@topol.io/editor*` 0.x / earlier 1.x alpha → 1.x (Email Editor v4)

> **Versions.** The target is `1.0.0-alpha.11` or later (`@topol.io/editor-react`
> `1.0.0-alpha.12` or later). Three starting points exist:
>
> | From | API today | Work |
> | --- | --- | --- |
> | `0.3.0` (`latest`) | Email Editor v3 | every step |
> | `1.0.0-alpha.1`–`.9` (editor-react up to `.10`) | Email Editor v3 | every step |
> | `1.0.0-alpha.10` (editor-react `.11`), `2.0.0-alpha.1`–`.3` | Email Editor v4, old names | steps 1–2, then *Coming from `1.0.0-alpha.10`* in the reference, then steps 8 and 10 |

The v4 API: `EmailEditor.init` takes `{ config, ...callbacks }` and returns an independent instance; nothing lives on `window` any more. Every callback receives zero arguments or one object. Reusable blocks are sections. Language mutations use `code`. `authorize.userId` is `authorize.entityId`. The `TopolPlugin` / `TopolEditor` names and the pre-1.0 type names are removed, not aliased. The Svelte wrapper needs Svelte 5 and takes callbacks as props.

[`references/reference.md`](references/reference.md) holds the version map, the differences per starting point, every rename table, the per-framework before/after code, loader rules, wrapper semantics, and the nonce and Cloud setup. [`references/landing-page-editor.md`](references/landing-page-editor.md) covers adding the Landing Page Editor and fixing Landing Page Editor code written against an earlier alpha.

## Steps

### 1. Inventory the integration

Find every touchpoint before editing anything:

```sh
grep -rnE "@topol\.io/editor|TopolPlugin|TopolEditor|EmailEditor|LandingPageEditor|VITE_TOPOL|topol-email-editor-id|topol-editor-id|topol-landing-page-editor-id|#editor|loader/build\.js|topol-lpe\.js" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git \
  --exclude=package-lock.json --exclude=yarn.lock --exclude=pnpm-lock.yaml
```

Record the installed version of each Topol package (it selects the row in the table above) and, for each hit, its kind: dependency, init/mount, method call, callback handler, config option, type import, CSS/DOM selector, loader script or env var. Note which framework wrapper (or plain core) is used, the framework's major version, and whether the app uses React StrictMode, SSR, or several editors per page.

If every Topol package is already at `1.0.0-alpha.11`+ (editor-react `1.0.0-alpha.12`+), there is nothing to migrate.

Done when every file that imports a Topol package is listed, and every identifier from the rename tables in `references/reference.md` that appears in the app has a line in the inventory.

### 2. Install the current 1.x release

**Svelte gate.** `@topol.io/editor-svelte` 1.x requires Svelte 5. If the app is on Svelte 4, stop and ask: upgrade Svelte first (a separate task), replace the wrapper with the core `@topol.io/editor` driven from `onMount`, or stay on `0.3`.

1.x is a prerelease, published on the `alpha` dist-tag while `latest` still points at `0.3.0`. Each package has its own number (editor-react is one ahead), so resolve them separately and pin exact versions, never `^`:

```sh
npm view @topol.io/editor dist-tags          # need 1.0.0-alpha.11 or later
npm view @topol.io/editor-react dist-tags    # need 1.0.0-alpha.12 or later (-vue / -svelte: .11+)
npm install @topol.io/editor-react@<resolved version>   # wrappers pull in the matching core
npm ls @topol.io/editor
```

Take the highest 1.x version among the tags (`alpha` today; a `beta` tag or `latest` once 1.x moves on). If no tag reaches those versions, stop and tell the user. If the app also depends on the core directly, pin it to the version the wrapper depends on (`npm view @topol.io/editor-react@<version> dependencies`). Use the app's own package manager (detect it from the lockfile). If the app pins `@topol.io/editor` 0.x deliberately (comment, renovate config), ask before bumping.

Done when `npm ls @topol.io/editor` shows a single 1.x entry at `1.0.0-alpha.11` or later.

### 3. Rewrite initialization and lifecycle

The `id` option and nested `callbacks` object are gone. `init` resolves to an instance before authentication; `render(container)` mounts the iframe; `onInit` reports successful authentication; `destroy()` runs on teardown, never right after `render`. The container must exist and have a height.

Branch by framework, following the matching before/after in `references/reference.md`:

- **Core**: `const editor = await EmailEditor.init({ config, ...callbacks }, { stage?, loaderUrl? }); await editor.render("#editor");` The default export and `TopolPlugin` are gone; import the named `EmailEditor`.
- **React**: `<EmailEditor options={config} onSave={...} ref={ref} onReady={...} />`. The default import and `TopolEditor` become the named `EmailEditor`.
- **Vue**: `<EmailEditor :options="config" :callbacks="callbacks" @on-save="..." @ready="..." />`. Handlers whose return value the editor must await (`onSave`, `onSaveAndClose`, `onEditorClose`, `getNonce`) go in the `callbacks` prop; events are notifications only.
- **Svelte** (Svelte 5): `<EmailEditor {options} onSave={(payload) => ...} onReady={(editor) => ...} />`. Every callback is a prop receiving the payload object; `on:` events, `event.detail`, the `callbacks` prop and `on:ready` are gone.

Done when no `id:` option, nested `callbacks:` key, `TopolPlugin` / `TopolEditor` import, or Svelte `on:on…` listener remains, and each mounted component owns exactly one instance.

### 4. Move method calls onto the instance

Replace every `TopolPlugin.x()` / `EmailEditor.x()` / `CoreEmailEditor.x()` / `window.TopolPlugin.x()` with a call on the instance from step 3 (React `ref.current?.getEditor()` or `onReady`; Vue template ref `getEditor()` or `@ready`; Svelte `onReady` or `bind:this` + `getEditor()`). All methods return promises; await `save()` where the host must observe completion. `save()` rejects with `err.code === "SAVE_IN_PROGRESS"` while another save runs.

Renames: `save('en')` → `save({ langCode: 'en' })`, `setSavedBlocks` → `setSavedSections`, `refreshSyncedRows` → `refreshSyncedSections`, `toggleBlocksAndStructuresVisibility` → `toggleHiddenElementsVisibility`, `toggleControlPanel` → `toggleSelectionPanel`. `getMutations()` resolves to `{ code, primary }[]`. Full table in `references/reference.md`.

Done when `TopolPlugin` and `window.TopolPlugin` no longer appear, and every renamed method is replaced.

### 5. Convert callbacks to object payloads

Rewrite each handler to take `()` or `({ ... })`, applying the callback table in `references/reference.md`. Renames: `onClose` → `onEditorClose`, `onError(type, message, body)` → `onEditorError({ source, message, detail })`, `onBlockSave/Remove/Edit` → `onSavedSectionSave/Remove/Edit`, `onEdittedWithoutSaveChanged(bool)` → `onUnsavedChangesChange({ hasUnsavedChanges })`, `updateTestingEmailAddresses(emails)` → `onTestingEmailsChange({ emails })`. Language payloads use `code`, not `lang`/`key`. Vue events that passed a bare value (`count`, `html`, `title`, …) now pass an object too.

The editor drops `onClose` and `onError` keys silently (they are reserved by the iframe bridge), so a leftover handler never fires. `onSave` that persists to a backend must return its promise; a rejection surfaces as a save error in the editor. Code that switches on `onEditorError`'s `source` must use `load`, not `templateLoad`.

Done when every handler in the inventory matches a current signature and none is named `onClose`, `onError`, or `onBlock*`.

### 6. Rename configuration options and types

Apply the config table in `references/reference.md`: `authorize.userId` → `authorize.entityId`, `savedBlocks` → `savedSections`, `premadeBlocks` (+ `premadeBlocksNoOverwrite`) → `premadeSections`, `syncedRowsEnable` → `syncedSectionsEnabled`, `topBarOptions` → `hideTopbarControls` (invert the list), `defaultTemplateSettings.langs[].key` → `code`. Remove the deleted `api` endpoints (`SAVE`, `LOAD`, `PREVIEW`, `AUTOSAVE`, `GET_AUTOSAVE`, `SYNCED_ROWS`, `TEST_EMAIL`, `IMAGE_EDITOR_UPLOAD`; `SAVED_BLOCKS` → `SAVED_SECTIONS`) and the removed options listed there. `entityId` must be a non-negative integer or match `[A-Za-z0-9_-]+`.

Rename the deprecated-but-working names in the same table too (`hideControlPanel`, `productMergetagTags`, the `defaultTemplateSettings` keys, …); they are removed in the next major.

Types: the config is `IEmailConfig` (or the wrapper alias `IReactEmailOptions` / `IVueEmailOptions` / `ISvelteEmailOptions`). `ITopolOptions` is now `IEmailOptions` and means the factory props `{ config, ...callbacks }`. `ISaveData`, `IErrorData`, `ISendTestData`, `ISavedBlock`, `ITopolPlugin` and the wrapper prop/ref types are renamed; apply the type table in `references/reference.md`.

Done when the project typecheck (`tsc`, `vue-tsc`, or `svelte-check`) reports zero errors in the inventory files.

### 7. Fix loader, environment, and layout

Build-time overrides are `VITE_TOPOL_V4_URL`, `VITE_TOPOL_V4_DEV_URL`, `VITE_TOPOL_V4_STAGING_URL`; old `VITE_TOPOL_URL` values are ignored. Prefer the runtime `loaderUrl` init option for previews. Remove any `<script src=".../loader/build.js">` the host page inserts itself; the package loads it and rejects host-inserted scripts. One loader URL per page; switching environments needs a reload.

The wrappers no longer render `#editor`, `#topol-editor-id`, `#topol-email-editor-id` or `#topol-landing-page-editor-id`, and their container is no longer `position: absolute`. Update CSS and DOM code that targeted those ids, and size the container (React `style`/`className`, Vue `class`/`style`, Svelte `class`/`style`).

Done when no v3 loader URL, `VITE_TOPOL_URL`, host-inserted loader script, or old container id remains in code, styles, env files, or HTML.

### 8. Landing Page Editor (only when present or asked)

Apps already using `LandingPageEditor` from an earlier alpha need the changes in `references/landing-page-editor.md`: object payloads and `onEditorError` / `onEditorClose` (code from `1.0.0-alpha.1`–`.8`), and the new wrapper container, promise-returning ref and renamed types (any earlier alpha). Apps on 0.3.0 get the Landing Page Editor as new functionality; add it only when asked.

### 9. Adopt nonce authentication or Topol Cloud (only when asked)

Nonce authentication is standalone: passing `getNonce` switches the editor from the public-key authorize call to short-lived `nc_` nonces that a host server endpoint creates with the Secret key for the signed-in user. There is no separate option, and every feature stays external. It works for both editors. Topol Cloud (`cloud.enabled`, Email Editor) is layered on top and requires `getNonce`. Follow the Nonce authentication and Topol Cloud sections of `references/reference.md`. Cloud-owned features silence `onSave` / `onTestSend` and ignore matching `api` endpoints.

### 10. Verify

1. Typecheck and build pass.
2. The leftover grep in `references/reference.md` returns nothing.
3. Run the app: `onInit` fires, a template loads, a save reaches the handler with the object payload, unmount logs no errors and no deprecation warnings from the editor. Under StrictMode or a double mount exactly one iframe exists.
4. Report every changed file and the behaviour changes the host must know: saves are awaited, `onInit` now means authenticated, a failed authentication requires destroying and recreating the editor, and the account's API key must work with Email Editor v4.

## Cannot be automated — always flag these

- **Backend endpoints** (apps coming from the v3 API that host `api.*` endpoints): CORS must allow `https://v4.email-assets.topol.io`, and autosaves moved to one `AUTOSAVES` URL (`GET AUTOSAVES`, `POST AUTOSAVES`, `GET AUTOSAVES/{key}`). Details: <https://docs.topol.io/email-editor/guide/v4-migration.html#backend-and-security-policy>. Name the endpoints the app configures and ask who owns the backend change.
- **Content Security Policy.** A page with a CSP must allow `v4.email-assets.topol.io` instead of the v3 domain in `script-src` and `frame-src`. The Landing Page Editor now loads `https://v1.page-assets.topol.io/loader/build.js` (was `topol-lpe.js`); a CSP that lists the script by full path needs the new one.
- **`onTestSend` handlers** coming from 0.3.0: `email` is `string | string[]`. A handler that feeds it to an API expecting one address needs a decision (send to all, or take the first). Ask, don't guess.
- **Layout.** Removing the old container ids and absolute positioning can silently move the editor. Show the CSS diff and ask the user to check the screen.
- **Templates and account.** Existing templates now open in Email Editor v4, and the API key must be enabled for v4. Tell the user to open and save one real template.
- **Svelte 4 apps.** The Svelte major upgrade is the user's decision (step 2).
- **Prerelease pin.** The user owns the decision to ship a prerelease to production. Flag it.

## Rules

- Never edit files under `node_modules/`.
- Do not upgrade unrelated dependencies while in the lockfile.
- Do not silence the migration's type errors with `any`; they mark the code that must change.
