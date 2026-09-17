---
name: topol-editors-upgrade-v2
description: 'Upgrade a host app from `@topol.io/editor*` 0.x or 1.x (Email Editor v3 runtime) to 2.x (npm major; Email Editor v4 runtime). Covers `@topol.io/editor`, `-react`, `-vue`, `-svelte`; the Landing Page Editor is untouched. Use when asked to upgrade or migrate Topol editor packages to 2.x or Email Editor v4, to port `TopolPlugin` / `TopolEditor` code to the `EmailEditor` instance API, when a 2.x install throws "Editor v4 requires { config: ... }" or callbacks stop firing after a bump, or to add nonce authentication (`getNonce`) or Topol Cloud (`cloud`) to an existing integration. For 0.x → 1.x only, use topol-editors-upgrade-v1.'
---

# Upgrade `@topol.io/editor*` 0.x/1.x → 2.x

> **Versions.** `2.x` is the npm major of `@topol.io/editor*`. It moves the
> Email Editor runtime from **v3** to **v4** and changes the API shape. The
> Landing Page Editor is not affected.

2.x targets Email Editor v4. `EmailEditor.init` takes `{ config, ...callbacks }` and returns an independent instance; nothing lives on `window` any more. Every callback receives zero arguments or one object. Reusable blocks are sections. Language mutations use `code`. Only a v4 loader works. `LandingPageEditor` is untouched.

`0.3.0` (`latest`) and `1.0.0-alpha.x` share the v3 contract; migrate both the same way. Everything below applies to the email editor only.

[`references/reference.md`](references/reference.md) holds every rename table, the per-framework before/after code, loader rules, wrapper semantics, and the Cloud setup.

## Steps

### 1. Inventory the integration

Find every touchpoint before editing anything:

```sh
grep -rnE "@topol\.io/editor|TopolPlugin|TopolEditor|EmailEditor|VITE_TOPOL|topol-email-editor-id|topol-editor-id|loader/build\.js" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git \
  --exclude=package-lock.json --exclude=yarn.lock --exclude=pnpm-lock.yaml
```

Record for each hit its kind: dependency, init/mount, method call, callback handler, config option, type import, loader script or env var. Note which framework wrapper (or plain core) is used and whether the app uses React StrictMode, SSR, or several editors per page.

Done when every file that imports a Topol package is listed, and every identifier from the rename tables in `references/reference.md` that appears in the app has a line in the inventory.

### 2. Install 2.x

Bump the core and every wrapper to the same 2.x version. Wrappers depend on the core, so an unbumped core leaves two copies in the lockfile.

```sh
npm view @topol.io/editor dist-tags   # 2.x prereleases are on `alpha`; pin the exact version
npm install @topol.io/editor@<2.x> @topol.io/editor-react@<2.x>   # or -vue / -svelte
npm ls @topol.io/editor
```

Done when `npm ls @topol.io/editor` shows a single 2.x entry.

### 3. Rewrite initialization and lifecycle

The `id` option and nested `callbacks` object are gone. `init` resolves to an instance before authentication; `render(container)` mounts the iframe; `onInit` reports successful authentication; `destroy()` runs on teardown, never right after `render`. The container must exist and have a height.

Branch by framework, following the matching before/after in `references/reference.md`:

- **Core**: `const editor = await EmailEditor.init({ config, ...callbacks }, { stage?, loaderUrl? }); await editor.render("#editor");`
- **React**: `<EmailEditor options={config} onSave={...} ref={ref} onReady={...} />`. Default import of `TopolEditor` becomes the named `EmailEditor`.
- **Vue**: `<EmailEditor :options="config" :callbacks="callbacks" @on-save="..." @ready="..." />`. Handlers whose return value the editor must await (`onSave`, `onSaveAndClose`, `getNonce`) go in the `callbacks` prop; events are notifications only.
- **Svelte**: `<EmailEditor {options} {callbacks} on:onSave={(e) => e.detail} on:ready={...} />`. The old `ITopolOptions` wrapper type is now `ISvelteOptions`.

Done when no `id:` option, nested `callbacks:` key, or `TopolPlugin` default import remains, and each mounted component owns exactly one instance.

### 4. Move method calls onto the instance

Replace every `TopolPlugin.x()` / `EmailEditor.x()` / `window.TopolPlugin.x()` with a call on the instance from step 3 (React `ref.current?.getEditor()` or `onReady`; Vue template ref `getEditor()` or `@ready`; Svelte `bind:this` + `getEditor()` or `on:ready`). All methods return promises; await `save()` where the host must observe completion.

Renames: `save('en')` → `save({ langCode: 'en' })`, `setSavedBlocks` → `setSavedSections`, `refreshSyncedRows` → `refreshSyncedSections`. `getMutations()` resolves to `{ code, primary }[]`. Full table in `references/reference.md`.

Done when `TopolPlugin.` and `window.TopolPlugin` no longer appear, and every renamed method is replaced.

### 5. Convert callbacks to object payloads

Rewrite each handler to take `()` or `({ ... })`, applying the callback table in `references/reference.md`. Renames: `onClose` → `onEditorClose`, `onError(type, message, body)` → `onEditorError({ source, message, detail })`, `onBlockSave/Remove/Edit` → `onSavedSectionSave/Remove/Edit`. Language payloads use `code`, not `lang`/`key`.

2.x drops `onClose` and `onError` keys silently (they are reserved by the iframe bridge), so a leftover handler never fires. `onSave` that persists to a backend must return its promise; a rejection surfaces as a save error in the editor.

Done when every handler in the inventory matches a 2.x signature and none is named `onClose`, `onError`, or `onBlock*`.

### 6. Rename configuration options

Apply the config table in `references/reference.md`: `savedBlocks` → `savedSections`, `premadeBlocks` (+ `premadeBlocksNoOverwrite`) → `premadeSections`, `syncedRowsEnable` → `syncedSectionsEnabled`, `topBarOptions` → `hideTopbarControls`, `defaultTemplateSettings.langs[].key` → `code`. Remove the deleted `api` endpoints (`SAVE`, `LOAD`, `PREVIEW`, `AUTOSAVE`, `GET_AUTOSAVE`, `SYNCED_ROWS`; `SAVED_BLOCKS` → `SAVED_SECTIONS`). `authorize.userId` must be a non-negative integer or match `[A-Za-z0-9_-]+`.

Type the config as `IEmailConfig`; `ITopolOptions` now means the factory props `{ config, ...callbacks }`.

Done when the project typecheck (`tsc`, `vue-tsc`, or `svelte-check`) reports zero errors in the inventory files.

### 7. Fix loader and environment

Build-time overrides are `VITE_TOPOL_V4_URL`, `VITE_TOPOL_V4_DEV_URL`, `VITE_TOPOL_V4_STAGING_URL`; old `VITE_TOPOL_URL` values are ignored. Prefer the runtime `loaderUrl` init option for previews. Remove any `<script src=".../loader/build.js">` the host page inserts itself; 2.x loads it and rejects host-inserted scripts. One loader URL per page; switching environments needs a reload.

Done when no v3 loader URL, `VITE_TOPOL_URL`, or host-inserted loader script remains in code, env files, or HTML.

### 8. Adopt nonce authentication or Topol Cloud (only when asked)

Nonce authentication is standalone: passing `getNonce` switches the editor from the public-key authorize call to short-lived `nc_` nonces minted by a host server endpoint for the signed-in user. There is no separate option, and every feature stays external. Topol Cloud (`cloud.enabled`) is layered on top and requires `getNonce`. Both need a `pk_` public key. Follow the Nonce authentication and Topol Cloud sections of `references/reference.md`. Cloud-owned features silence `onSave` / `onTestSend` and ignore matching `api` endpoints.

### 9. Verify

1. Typecheck and build pass.
2. The leftover grep in `references/reference.md` returns nothing.
3. Run the app: `onInit` fires, a template loads, a save reaches the handler with the object payload, unmount logs no errors. Under StrictMode or a double mount exactly one iframe exists.
4. Report every changed file and the behaviour changes the host must know: saves are awaited, `onInit` now means authenticated, and a failed authentication requires destroying and recreating the editor.
