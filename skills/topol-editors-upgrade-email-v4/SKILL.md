---
name: topol-editors-upgrade-email-v4
description: 'Upgrade a host app that embeds the Topol Email Editor with the plain script tag from Email Editor v3 (`v3.email-assets.topol.io/loader/build.js`, `TopolPlugin.init(TOPOL_OPTIONS)`) to v4 (`v4.email-assets.topol.io/loader/build.js`, `TopolEmailEditor({ config, ...callbacks })` + `render()`). Covers the instance API, object callback payloads, renamed options and `textOverride` keys, changed defaults, the host backend (`AUTOSAVES` endpoint contract, CORS) and CSP. Use when asked to upgrade or migrate a script-tag / CDN / non-npm Topol Email Editor integration to v4, when `TopolPlugin` is undefined after switching the loader, or when callbacks stop firing after a v4 switch. For apps using the `@topol.io/editor*` npm packages use topol-editors-upgrade-v1; for a first-time integration use topol-editors-integration.'
---

# Upgrade a script-tag Email Editor integration from v3 to v4

> **Scope.** This skill is for pages that load the editor with
> `<script src="https://v3.email-assets.topol.io/loader/build.js">` and call
> `TopolPlugin.init(...)`. Apps that import `@topol.io/editor*` from npm use
> `topol-editors-upgrade-v1` instead. Template JSON is compatible: v4 opens v3
> templates as they are.

The v4 API: the loader defines a global factory `TopolEmailEditor({ config, ...callbacks })` that returns an editor instance; `instance.render(container)` mounts the iframe. Options live under `config`, callbacks are top-level props next to it. Every method returns a Promise. Every callback receives zero arguments or one object. Reusable blocks are sections. Language mutations use `code`.

[`references/reference.md`](references/reference.md) holds the before/after code, every rename table, the changed defaults, the backend contract, and the behaviour changes.

## Steps

### 1. Inventory the integration

Find every touchpoint before editing anything. The editor is often configured in server-rendered templates as well as JS, so search every source type:

```sh
grep -rnE "email-assets\.topol\.io|TopolPlugin|TOPOL_OPTIONS|TopolEmailEditor|savedBlocks|premadeBlocks|topBarOptions|onBlock(Save|Remove|Edit)|onError|onClose|AUTOSAVE|GET_AUTOSAVE|textOverride" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git --exclude-dir=vendor
```

Record for each hit its kind: loader script, options object, init call, method call, callback handler, `api.*` endpoint, `textOverride` key, CSP/CORS config, or backend route. Note whether several editors share a page and whether a backend serves `api.*` endpoints (autosaves, saved sections, comments).

If the page already loads `v4.email-assets.topol.io` and calls `TopolEmailEditor(`, there is nothing to migrate.

Done when every file that touches the editor, its options, its callbacks or its `api.*` endpoints is listed.

### 2. Switch the loader

Replace the v3 loader URL with `https://v4.email-assets.topol.io/loader/build.js`. Load it once per page: a second copy throws `Can not register multiple components with the same tag`. One loader can create several editors.

Done when no `v3.email-assets.topol.io` URL remains in HTML, templates, or JS.

### 3. Rewrite initialization

Follow the before/after in `references/reference.md`:

- Move every option into `config`. Remove `id`.
- Move every callback out of `callbacks` to the top level, next to `config`.
- Replace `TopolPlugin.init(TOPOL_OPTIONS)` with `const editor = TopolEmailEditor({ config, ...callbacks }); editor.render("#app");`. The container must exist and have an explicit width and height.
- Keep the instance where every later method call can reach it (module variable, component state, `window` property the app already uses).

`authorize.userId` must be a non-negative integer or match `[A-Za-z0-9_-]+`. If the app passes emails or other IDs, flag it (see "Cannot be automated").

Done when no `TopolPlugin.init`, `id:` option or nested `callbacks:` key remains.

### 4. Move method calls onto the instance

Replace every `TopolPlugin.x()` / `window.TopolPlugin.x()` with `editor.x()`. Renames: `destroy()` → `close()`, `setSavedBlocks` → `setSavedSections`, `save('en')` → `save({ langCode: 'en' })`. `getMutations()` now resolves to `{ code, primary }[]` (it still fires `onGetMutations`). All methods return promises; `await editor.save()` where the page must observe completion.

Done when `TopolPlugin` no longer appears anywhere.

### 5. Convert callbacks to object payloads

Rewrite each handler to take `()` or `({ ... })`, applying the callback table in `references/reference.md`. Renames: `onClose` → `onEditorClose`, `onError(type, message, responseBody)` → `onEditorError({ source, message, detail })`, `onBlockSave/Remove/Edit` → `onSavedSectionSave({ section })` / `onSavedSectionRemove({ id })` / `onSavedSectionEdit({ id })`. Language payloads use `code`, not `lang`.

A callback under its v3 name is never called. `onClose` and `onError` are worse: they are the iframe library's own lifecycle props in v4 and fire on its events, so a leftover handler runs at the wrong time. `onSave` that persists to a backend should return its promise so the editor's saving indicator waits for it.

Done when every handler matches a v4 signature and none is named `onClose`, `onError`, or `onBlock*`.

### 6. Rename options

Apply the options table in `references/reference.md`: `savedBlocks` → `savedSections`, `premadeBlocks` → `premadeSections` (drop `premadeBlocksNoOverwrite`), `topBarOptions` → `hideTopbarControls` (invert the list), `defaultTemplateSettings.langs[].key` → `code`, renamed `textOverride` keys, and remove `api.AUTOSAVE`, `api.GET_AUTOSAVE`, `api.SAVE`, `api.LOAD`. Unknown options are ignored silently, so a missed rename produces no error.

Then check the changed defaults (`smartMergeTags.enabled`, `mobileFirstEnabled`, `premadeTemplates`, `showControlPanelExpand` are now `true`). Ask the user whether to keep the new defaults or pin the v3 values; do not decide for them.

Done when no v3-only option name remains and the defaults decision is recorded.

### 7. Update the backend and security policy (only when present)

If the app serves any `api.*` endpoints:

- Allow CORS from `https://v4.email-assets.topol.io`.
- Make the autosave route answer `GET AUTOSAVES` (list), `POST AUTOSAVES` (create) and `GET AUTOSAVES/{key}` (read one), replacing v3's `POST AUTOSAVE` and `GET GET_AUTOSAVE/{key}`. Bodies and params are unchanged.
- Let the `SAVED_SECTIONS` list honour `type=folder` (return only folders).
- If the backend relied on the undocumented v3 `api.SAVE` POST after each save, move that persistence into `onSave`.

If the page sets a Content Security Policy or Permissions-Policy, replace `v3.email-assets.topol.io` with `v4.email-assets.topol.io` in `script-src`, `frame-src` and feature allowlists.

Done when no backend route, CORS list or CSP still names the v3 host or the `AUTOSAVE` / `GET_AUTOSAVE` endpoints.

### 8. Verify

1. The leftover grep in `references/reference.md` returns nothing.
2. The app builds and its own tests pass.
3. Open the page: `onInit` fires, an existing template loads, editing then saving reaches `onSave` with `{ json, html }`, an autosave is created and listed, closing the editor works.
4. Report every changed file and the behaviour changes the host must know (see `references/reference.md` → Behaviour changes).

## Cannot be automated — always flag these

- **`userId` format.** IDs with `@`, `.` or other characters now fail init. Mapping them changes where users' images are stored and how they are billed; the user decides the mapping.
- **Changed defaults.** Smart merge tags, mobile-first, premade templates and the control-panel expand button now default to on. Product decision; ask.
- **Backend deploy order.** The autosave route change must be live before the v4 page ships. Tell the user.
- **Account and templates.** The API key must be enabled for Email Editor v4. Tell the user to open and save one real template.

## Rules

- Do not touch template JSON: v4 reads v3 templates as they are (`langs[].key` inside templates stays `key`).
- Do not add the npm packages to a script-tag app as part of this upgrade.
- Do not rename `onClose` / `onError` handlers of other libraries; only the ones passed to the Topol editor.
