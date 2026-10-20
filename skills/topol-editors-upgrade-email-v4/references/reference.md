# Reference: script-tag Email Editor v3 → v4

Source of truth: the v4 loader (`apps/email-editor/zoid/init.js`), `apps/email-editor/types/TopolPlugin.ts` and `apps/email-editor/src/schemas/pluginOptions/pluginOptionsSchema.ts` in the Topol editors repo, and the upgrade guide at https://docs.topol.io/email-editor/guide/v3-to-v4-migration.

## Before / after

```html
<!-- v3 -->
<script src="https://v3.email-assets.topol.io/loader/build.js"></script>
<div id="app" style="height: 100vh"></div>
<script>
  const TOPOL_OPTIONS = {
    id: "#app",
    authorize: { apiKey: "YOUR_API_KEY", userId: "user-123" },
    savedBlocks: savedBlocks,
    topBarOptions: ["undoRedo", "save", "previewSize"],
    callbacks: {
      onSave(json, html) {
        saveTemplate(json, html);
      },
      onBlockSave(block) {
        createBlock(block);
      },
      onError(type, message, responseBody) {
        report(type, message);
      },
      onClose() {
        leave();
      },
    },
  };
  TopolPlugin.init(TOPOL_OPTIONS);
  TopolPlugin.load(template);
  // later
  TopolPlugin.setSavedBlocks(savedBlocks);
  TopolPlugin.save("en");
  TopolPlugin.destroy();
</script>
```

```html
<!-- v4 -->
<script src="https://v4.email-assets.topol.io/loader/build.js"></script>
<div id="app" style="height: 100vh"></div>
<script>
  const editor = TopolEmailEditor({
    config: {
      authorize: { apiKey: "YOUR_API_KEY", userId: "user-123" },
      savedSections: savedBlocks,
      // was an allowlist of buttons to show; now the buttons to hide
      hideTopbarControls: ["saveAndClose", "changePreview", "previewTestMail", "changeLanguage", "translate"],
    },
    onSave({ json, html }) {
      return saveTemplate(json, html); // returned promise keeps the save spinner until it settles
    },
    onSavedSectionSave({ section }) {
      createBlock(section);
    },
    onEditorError({ source, message, detail }) {
      report(source, message);
    },
    onEditorClose() {
      leave();
    },
  });
  editor.render("#app");
  editor.load(template); // queued until the editor is ready
  // later
  editor.setSavedSections(savedBlocks);
  editor.save({ langCode: "en" });
  editor.close();
</script>
```

## Initialization

| v3 | v4 |
| --- | --- |
| `TopolPlugin.init(options)` on the global singleton | `TopolEmailEditor({ config, ...callbacks })` returns an instance; call `render()` on it |
| `id: "#app"` option | `editor.render("#app")` or `editor.render(element)`; a missing element throws `Can not find element` |
| Options at the top level | Options under `config` (required; missing `config` throws `Expected prop "config" to be defined`) |
| `callbacks: { … }` inside the options | Callbacks as top-level props next to `config`; ones left in `config` are stripped |
| Loader may be included twice (v3 skipped re-creation) | Loader once per page; a second copy throws `Can not register multiple components with the same tag` |
| `userId`: any string or number | Non-negative integer, or string matching `[A-Za-z0-9_-]+`; otherwise init fails |

Method calls made before the editor is ready are queued and resolve after initialization. If authentication or init fails, the host gets `onEditorError({ source: "authorize" })` and then `onEditorError({ source: "init" })`; queued calls never settle. Recover with `editor.close()` and a new `TopolEmailEditor(...)`.

## Methods

All methods return Promises.

| v3 | v4 |
| --- | --- |
| `TopolPlugin.load(json)` | `editor.load(json)`; resolves once applied, resets the unsaved flag |
| `TopolPlugin.save()` / `save(lang)` | `editor.save()` / `save({ langCode })`; a bare string is ignored; rejects with `SaveInProgressError` while another save runs; other failures resolve and report `onEditorError({ source: "save" })` |
| `TopolPlugin.destroy()` | `editor.close()` |
| `TopolPlugin.setSavedBlocks(list)` | `editor.setSavedSections(list)` |
| `TopolPlugin.getMutations()` → result via `onGetMutations` | `editor.getMutations()` → `Promise<{ code, primary }[]>`, still fires `onGetMutations` |
| `createLanguage / deleteLanguage / selectLanguage / setPrimaryLanguage(lang)` | Same names; argument is the language code |
| — | `translateLanguage(code, sourceCode?)`, `translateAllLanguages()` |

Unchanged names: `togglePreview`, `togglePreviewSize`, `toggleDarkMode`, `chooseFile`, `undo`, `redo`, `setPreviewHTML`, `createNotification`, `setActiveMembers`, `changeEmailToMobile`, `changeEmailToDesktop`, `toggleBlocksAndStructuresVisibility`, `updateCustomBlockContent`, `refreshComments`, `refreshSyncedSections`, `openPremadeTemplatesSelection`, `updateApiAuthorizationHeader`, `setTemplateName`, `updateOptions`, `setMergeTags`, `toggleChatAI`, `toggleAutosaves`, `toggleComments`, `toggleControlPanel`, `updateTemplate`.

## Callbacks

| v3 | v4 |
| --- | --- |
| `onSave(json, html, mutations, syncedSections)` | `onSave({ json, html, mutations, syncedSections })`, may return a Promise |
| `onSaveAndClose(json, html, mutations, syncedSections)` | `onSaveAndClose({ json, html, mutations, syncedSections })`, may return a Promise |
| `onTestSend(email, json, html)` | `onTestSend({ email, json, html })` |
| `onBannerClick(json, html)` | `onBannerClick({ json, html })` |
| `onImageDelete(items)` | `onImageDelete({ items })`, item shape unchanged |
| `onUndoChange(count)` / `onRedoChange(count)` | `({ count })` |
| `onPreview(html)` | `onPreview({ html })` |
| `onAlert(notification)` | `onAlert({ notification })`; still fires only with `disableAlerts: true` |
| `onEdittedWithoutSaveChanged(bool)` | `onEdittedWithoutSaveChanged({ value })` |
| `onBlocksAndStructuresVisibilityChange(bool)` | `onBlocksAndStructuresVisibilityChange({ value })` |
| `onOpenCustomBlockDialog(block)` | `onOpenCustomBlockDialog({ block })` |
| `onTemplateRename(title)` | `onTemplateRename({ name })` |
| `updateTestingEmailAddresses(emails)` | `updateTestingEmailAddresses({ emails })` |
| `onLanguageCreated / onLanguageDeleted / onPrimaryLanguageChanged(lang, mutations)` | `({ code, mutations })` |
| `onLanguageSelected(lang)` | `onLanguageSelected({ code })` |
| `onGetMutations(mutations)` | `onGetMutations({ mutations })` |
| `onError(type, message, responseBody)` | `onEditorError({ source, message, detail })` |
| `onClose()` | `onEditorClose()` |
| `onBlockSave(block)` | `onSavedSectionSave({ section })`; `section` carries `definition` and `translation` |
| `onBlockRemove(blockId)` | `onSavedSectionRemove({ id })`; `id` is a number or a string |
| `onBlockEdit(blockId)` | `onSavedSectionEdit({ id })`; `id` is a number or a string |
| `onInit()`, `onLoaded()`, `onOpenFileManager()`, `onTemplateUpdated()`, `onPreviewClose()` | Unchanged |

`mutations` items change from `{ lang, primary }` to `{ code, primary }`.

`onEditorError` sources include `authorize`, `init`, `save`, and `nonceAuth` (with `getNonce`).

`onClose` and `onError` are reserved by the iframe library (zoid) in v4 and bind to its lifecycle events, not the editor's. Undeclared callback names are passed through without error and never called.

## Options

| v3 | v4 |
| --- | --- |
| `savedBlocks` | `savedSections`, same item shape `{ id, name, img?, definition, translation?, children? }` |
| `premadeBlocks` | `premadeSections`, same value shapes |
| `premadeBlocksNoOverwrite` | Removed (it was unused in v3) |
| `topBarOptions` (buttons to show) | `hideTopbarControls` (buttons to hide); invert against `undoRedo`, `save`, `saveAndClose`, `changePreview`, `previewSize`, `previewTestMail`, `changeLanguage`, `translate` |
| `defaultTemplateSettings.langs[].key` | `defaultTemplateSettings.langs[].code` (template JSON itself keeps `langs[].key`) |
| `api.AUTOSAVE`, `api.GET_AUTOSAVE` | Removed; `api.AUTOSAVES` handles list, create and read |
| `api.SAVE`, `api.LOAD` | Removed; persist in `onSave`, load with `editor.load()` |
| `id`, `callbacks` | Removed (see Initialization) |

Unknown option keys are stripped without error.

### `textOverride` keys

| v3 | v4 |
| --- | --- |
| `saved_blocks*` | `saved_sections*` |
| `premade_blocks` | `premade_sections` |
| `save_as_block` | `save_as_section` |
| `synced_rows` | `synced_sections` |
| `*_synced_row*` | `*_synced_section*` |

### Changed defaults

| Option | v3 | v4 |
| --- | --- | --- |
| `smartMergeTags.enabled` | `false` | `true` |
| `mobileFirstEnabled` | `false` | `true` |
| `premadeTemplates` | `false` | `true` |
| `showControlPanelExpand` | `false` | `true` |

The control panel also starts hidden when the iframe is narrower than 1024 px.

## Backend contract

| Endpoint | v3 | v4 |
| --- | --- | --- |
| CORS origin | `https://v3.email-assets.topol.io` | `https://v4.email-assets.topol.io` |
| List autosaves | `GET AUTOSAVES` | `GET AUTOSAVES` |
| Create autosave | `POST AUTOSAVE` | `POST AUTOSAVES` |
| Read autosave | `GET GET_AUTOSAVE/{key}` | `GET AUTOSAVES/{key}` |
| Saved sections | `SAVED_SECTIONS` | Same URLs and envelopes; list may send `type=folder` (return folders only) |
| Comments | `CONVERSATIONS`, `COMMENTS` | Unchanged |
| After each save | Optional `POST api.SAVE` with `{ …params, html, json }` | Removed; persist in `onSave` |

Autosave params and bodies (`key`, `hostname`, `templateId`, `uuid`, `definition`, `current_user_id`) are unchanged, and any 2xx is accepted. Before the first autosave of a session, v4 lists autosaves, reads the newest, and may create one extra "initial state" autosave. Autosave is skipped while an explicit save is running.

## Host page

- Container: explicit width and height; the iframe fills 100% × 100%.
- The iframe requests `fullscreen`, `clipboard-read`, `clipboard-write` and `local-network-access`, scoped to the v4 origin.
- CSP / Permissions-Policy: replace `v3.email-assets.topol.io` with `v4.email-assets.topol.io` in `script-src`, `frame-src` and feature allowlists. `api.topol.io` is unchanged.

## Behaviour changes

- **Awaited saves.** A Promise returned from `onSave` / `onSaveAndClose` keeps the saving indicator up until it settles. A rejection shows a saving error and reports `onEditorError({ source: "save" })`. After 30 s the editor stops waiting and shows an "unconfirmed" notice without `onEditorError`.
- **Unsaved flag.** It clears only after `onSave` settles and only if the template did not change meanwhile; v3 cleared it after rendering.
- **Save & Close without `onSaveAndClose`.** v4 calls `onSave` and then closes the editor (removes the iframe); v3 did nothing.
- **Window-bar close without `onEditorClose`.** v4 closes the editor itself; v3 needed the callback. If `onEditorClose` rejects, the editor also closes itself.
- **Hostname for authorization.** Taken from the parent page through the iframe bridge, so a strict `Referrer-Policy` no longer breaks authorization.

## Optional v4 features (only when asked)

- `getNonce()`: nonce authentication with short-lived `nc_` tokens minted by the host backend from the Secret key via Topol `POST /auth/nonce`.
- `cloud: { enabled, templateId?, features? }`: Topol Cloud storage; requires `getNonce`.
- `countdown` options and `api.COUNTDOWN_PRESETS`: Countdown block add-on.
- `disableShortcuts`, `linkTracking: { provider: "sparkpost" }`, `saveAndCloseButton: "dropdown" | "separate"`, `startWithOpenElementsPanel`.

## Leftover grep (must return nothing after migration)

```sh
grep -rnE "v3\.email-assets\.topol\.io|TopolPlugin|\bid: *['\"]#|callbacks: *\{|savedBlocks|setSavedBlocks|premadeBlocks|topBarOptions|onBlockSave|onBlockRemove|onBlockEdit|\bonError\b|\bonClose\b|\bAUTOSAVE\b|GET_AUTOSAVE|saved_blocks|premade_blocks|save_as_block|synced_row" . \
  --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git --exclude-dir=vendor
```

Hits for `onError` / `onClose` that belong to other libraries are fine; only the ones passed to the Topol editor must go.
