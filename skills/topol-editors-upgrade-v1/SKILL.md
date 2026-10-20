---
name: topol-editors-upgrade-v1
description: 'Upgrade a host app from `@topol.io/editor*` 0.x or an early 1.x alpha (Email Editor v3 API) to 1.x from `1.0.0-alpha.10` (Email Editor v4 API). Covers `@topol.io/editor`, `-react`, `-vue`, `-svelte`; the Landing Page Editor API is unchanged and becomes available to 0.x apps. Use when asked to upgrade, migrate, or bump Topol editor packages to 1.x or Email Editor v4, to port `TopolPlugin` / `TopolEditor` code to the `EmailEditor` instance API, when an install throws "Editor v4 requires { config: ... }", callbacks stop firing, or "no default export" / `ITopolOptions` type errors appear after a Topol bump, or to add nonce authentication (`getNonce`) or Topol Cloud (`cloud`) to an existing integration. For a first-time integration use topol-editors-integration.'
---

# Upgrade `@topol.io/editor*` 0.x / early 1.x alpha → 1.x (Email Editor v4)

> **Versions.** From `1.0.0-alpha.10` (`@topol.io/editor-react` `1.0.0-alpha.11`)
> the 1.x line embeds Email Editor **v4** and changes the API shape. `0.3.0`
> (`latest`) and `1.0.0-alpha.1`–`.9` (editor-react up to `.10`) use the v3 API
> and migrate the same way. The Landing Page Editor API is not affected.

The v4 API: `EmailEditor.init` takes `{ config, ...callbacks }` and returns an independent instance; nothing lives on `window` any more. Every callback receives zero arguments or one object. Reusable blocks are sections. Language mutations use `code`. Only a v4 loader works. `LandingPageEditor` is untouched.

[`references/reference.md`](references/reference.md) holds the version map, the extra 0.3.0 differences, every rename table, the per-framework before/after code, loader rules, wrapper semantics, and the Cloud setup. [`references/landing-page-editor.md`](references/landing-page-editor.md) covers adding the Landing Page Editor and fixing Landing Page Editor code from `1.0.0-alpha.1`–`.8`.

## Steps

### 1. Inventory the integration

Find every touchpoint before editing anything:

```sh
grep -rnE "@topol\.io/editor|TopolPlugin|TopolEditor|EmailEditor|LandingPageEditor|VITE_TOPOL|topol-email-editor-id|topol-editor-id|#editor|loader/build\.js" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git \
  --exclude=package-lock.json --exclude=yarn.lock --exclude=pnpm-lock.yaml
```

Record the installed version of each Topol package and, for each hit, its kind: dependency, init/mount, method call, callback handler, config option, type import, CSS/DOM selector, loader script or env var. Note which framework wrapper (or plain core) is used and whether the app uses React StrictMode, SSR, or several editors per page.

If every Topol package is already at `1.0.0-alpha.10`+ (editor-react `1.0.0-alpha.11`+), there is nothing to migrate.

Done when every file that imports a Topol package is listed, and every identifier from the rename tables in `references/reference.md` that appears in the app has a line in the inventory.

### 2. Install the 1.x v4 release

1.x is a prerelease on the `alpha` dist-tag; `latest` still points at `0.3.0`. Each package has its own number (editor-react is one ahead), so resolve them separately and pin exact versions, never `^`:

```sh
npm view @topol.io/editor dist-tags          # alpha must be 1.0.0-alpha.10 or later
npm view @topol.io/editor-react dist-tags    # alpha must be 1.0.0-alpha.11 or later (or -vue / -svelte: .10+)
npm install @topol.io/editor-react@<alpha version>   # wrappers pull in the matching core
npm ls @topol.io/editor
```

If `alpha` points below `1.0.0-alpha.10` (editor-react `1.0.0-alpha.11`), the 1.x v4 release is not published yet: stop and tell the user. If the app also depends on the core directly, pin it to the version the wrapper depends on (`npm view @topol.io/editor-react@<version> dependencies`). Use the app's own package manager (detect it from the lockfile). If the app pins `@topol.io/editor` 0.x deliberately (comment, renovate config), ask before bumping.

Done when `npm ls @topol.io/editor` shows a single 1.x entry at `1.0.0-alpha.10` or later.

### 3. Rewrite initialization and lifecycle

The `id` option and nested `callbacks` object are gone. `init` resolves to an instance before authentication; `render(container)` mounts the iframe; `onInit` reports successful authentication; `destroy()` runs on teardown, never right after `render`. The container must exist and have a height.

Branch by framework, following the matching before/after in `references/reference.md`:

- **Core**: `const editor = await EmailEditor.init({ config, ...callbacks }, { stage?, loaderUrl? }); await editor.render("#editor");`
- **React**: `<EmailEditor options={config} onSave={...} ref={ref} onReady={...} />`. The 0.3.0 default import of `TopolEditor` becomes the named `EmailEditor`.
- **Vue**: `<EmailEditor :options="config" :callbacks="callbacks" @on-save="..." @ready="..." />`. Handlers whose return value the editor must await (`onSave`, `onSaveAndClose`, `getNonce`) go in the `callbacks` prop; events are notifications only.
- **Svelte**: `<EmailEditor {options} {callbacks} on:onSave={(e) => e.detail} on:ready={...} />`. The old `ITopolOptions` wrapper type is now `ISvelteOptions`.

Done when no `id:` option, nested `callbacks:` key, or `TopolPlugin` default import remains, and each mounted component owns exactly one instance.

### 4. Move method calls onto the instance

Replace every `TopolPlugin.x()` / `EmailEditor.x()` / `CoreEmailEditor.x()` / `window.TopolPlugin.x()` with a call on the instance from step 3 (React `ref.current?.getEditor()` or `onReady`; Vue template ref `getEditor()` or `@ready`; Svelte `bind:this` + `getEditor()` or `on:ready`). All methods return promises; await `save()` where the host must observe completion.

Renames: `save('en')` → `save({ langCode: 'en' })`, `setSavedBlocks` → `setSavedSections`, `refreshSyncedRows` → `refreshSyncedSections`. `getMutations()` resolves to `{ code, primary }[]`. Full table in `references/reference.md`.

Done when `TopolPlugin.` and `window.TopolPlugin` no longer appear, and every renamed method is replaced.

### 5. Convert callbacks to object payloads

Rewrite each handler to take `()` or `({ ... })`, applying the callback table in `references/reference.md`. Renames: `onClose` → `onEditorClose`, `onError(type, message, body)` → `onEditorError({ source, message, detail })`, `onBlockSave/Remove/Edit` → `onSavedSectionSave/Remove/Edit`. Language payloads use `code`, not `lang`/`key`. Vue and Svelte events that passed a bare value (`count`, `html`, `title`, …) now pass an object too.

The editor drops `onClose` and `onError` keys silently (they are reserved by the iframe bridge), so a leftover handler never fires. `onSave` that persists to a backend must return its promise; a rejection surfaces as a save error in the editor.

Done when every handler in the inventory matches a v4 signature and none is named `onClose`, `onError`, or `onBlock*`.

### 6. Rename configuration options

Apply the config table in `references/reference.md`: `savedBlocks` → `savedSections`, `premadeBlocks` (+ `premadeBlocksNoOverwrite`) → `premadeSections`, `syncedRowsEnable` → `syncedSectionsEnabled`, `topBarOptions` → `hideTopbarControls` (invert the list), `defaultTemplateSettings.langs[].key` → `code`. Remove the deleted `api` endpoints (`SAVE`, `LOAD`, `PREVIEW`, `AUTOSAVE`, `GET_AUTOSAVE`, `SYNCED_ROWS`; `SAVED_BLOCKS` → `SAVED_SECTIONS`). `authorize.userId` must be a non-negative integer or match `[A-Za-z0-9_-]+`.

Type the config as `IEmailConfig` (or the wrapper alias `IReactEmailOptions` / `IVueEmailOptions` / `ISvelteOptions`); `ITopolOptions` now means the factory props `{ config, ...callbacks }`.

Done when the project typecheck (`tsc`, `vue-tsc`, or `svelte-check`) reports zero errors in the inventory files.

### 7. Fix loader, environment, and layout

Build-time overrides are `VITE_TOPOL_V4_URL`, `VITE_TOPOL_V4_DEV_URL`, `VITE_TOPOL_V4_STAGING_URL`; old `VITE_TOPOL_URL` values are ignored. Prefer the runtime `loaderUrl` init option for previews. Remove any `<script src=".../loader/build.js">` the host page inserts itself; the package loads it and rejects host-inserted scripts. One loader URL per page; switching environments needs a reload.

The wrappers no longer render `#editor`, `#topol-editor-id`, or `#topol-email-editor-id`, and their container is no longer `position: absolute`. Update CSS and DOM code that targeted those ids, and size the container (React `style`/`className`, Vue `class`/`style`, Svelte parent element).

Done when no v3 loader URL, `VITE_TOPOL_URL`, host-inserted loader script, or old container id remains in code, styles, env files, or HTML.

### 8. Landing Page Editor (only when present or asked)

Apps already using `LandingPageEditor` from `1.0.0-alpha.1`–`.8` must move its callbacks to object payloads and `onEditorError` / `onEditorClose`. Apps on 0.3.0 get it as new functionality; add it only when asked. Both are in `references/landing-page-editor.md`.

### 9. Adopt nonce authentication or Topol Cloud (only when asked)

Nonce authentication is standalone: passing `getNonce` switches the editor from the public-key authorize call to short-lived `nc_` nonces minted by a host server endpoint for the signed-in user. There is no separate option, and every feature stays external. Topol Cloud (`cloud.enabled`) is layered on top and requires `getNonce`. Both need a `pk_` public key. Follow the Nonce authentication and Topol Cloud sections of `references/reference.md`. Cloud-owned features silence `onSave` / `onTestSend` and ignore matching `api` endpoints.

### 10. Verify

1. Typecheck and build pass.
2. The leftover grep in `references/reference.md` returns nothing.
3. Run the app: `onInit` fires, a template loads, a save reaches the handler with the object payload, unmount logs no errors. Under StrictMode or a double mount exactly one iframe exists.
4. Report every changed file and the behaviour changes the host must know: saves are awaited, `onInit` now means authenticated, a failed authentication requires destroying and recreating the editor, and the account's API key must work with Email Editor v4.

## Cannot be automated — always flag these

- **`onTestSend` handlers** coming from 0.3.0: `email` is `string | string[]`. A handler that feeds it to an API expecting one address needs a decision (send to all, or take the first). Ask, don't guess.
- **Layout.** Removing the old container ids and absolute positioning can silently move the editor. Show the CSS diff and ask the user to check the screen.
- **Templates and account.** Existing templates now open in Email Editor v4, and the API key must be enabled for v4. Tell the user to open and save one real template.
- **Prerelease pin.** The user owns the decision to ship a prerelease to production. Flag it.

## Rules

- Never edit files under `node_modules/`.
- Do not upgrade unrelated dependencies while in the lockfile.
- Do not silence the migration's type errors with `any`; they mark the code that must change.
