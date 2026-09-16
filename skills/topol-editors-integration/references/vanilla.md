# Plain JS/TS — `@topol.io/editor`

The core package. Use it when there is no supported framework wrapper (vanilla
JS/TS, Angular, Vue 2, Svelte for the LPE) or when you need imperative control.

```bash
npm install @topol.io/editor@<resolved-v1-version>
```

ESM and CJS builds are both published; types ship with the package.

## Email Editor

```html
<div id="editor" style="position: absolute; width: 100%; height: 100%"></div>
```

```ts
import { EmailEditor, type ITopolOptions } from '@topol.io/editor';

const options: ITopolOptions = {
  id: '#editor',                       // required: CSS selector for the container
  authorize: { apiKey: TOPOL_API_KEY, userId: currentUser.id },
  callbacks: {
    onSave(json, html, mutations, syncedSections) {
      // TODO: persist the template — application decision
    },
    onError(type, message, responseBody) {
      console.error(type, message, responseBody);
    },
  },
};

await EmailEditor.init(options);              // resolves when the loader is ready
// await EmailEditor.init(options, { stage: 'production' });   // stage is optional
```

- `init()` returns a promise — `await` it (or `.catch()`), otherwise a failed
  loader fetch becomes an unhandled rejection.
- The editor is a **singleton**: one `init()` per page. Call `EmailEditor.destroy()`
  before re-initializing.
- Callbacks live under `callbacks`, unlike the LPE. See `options.md` for the full
  list and for the imperative methods (`save`, `load`, `undo`, `updateOptions`, …).

## Landing Page Editor

Instance-based, and the shape differs from the Email Editor in two ways worth
reading twice: config is nested under `config`, and callbacks sit at the **top
level** next to it.

```html
<div id="lpe" style="position: absolute; width: 100%; height: 100%"></div>
```

```ts
import { LandingPageEditor, type ILandingPageEditorInstance } from '@topol.io/editor';

const instance: ILandingPageEditorInstance = await LandingPageEditor.init({
  config: {
    authorize: { apiKey: TOPOL_API_KEY, userId: currentUser.id },
  },
  onSave(json, html) {/* TODO: persist */},
  onLoaded() {},
});

instance.render('#lpe');       // nothing renders until this runs
instance.load(templateJson);   // optional
// instance.save();  → triggers onSave
// instance.destroy();
```

The LPE config is validated with Zod at runtime, so a misspelled or wrongly-typed
option throws instead of being ignored. Its `onSave` takes two arguments, not four.

## Angular

Init from `ngAfterViewInit` (the container must exist in the DOM) and call
`EmailEditor.destroy()` from `ngOnDestroy`. Keep the container element outside any
`*ngIf` that could tear it down while the editor is live. Nothing else is special —
it is the vanilla flow above.

## Vue 2

`@topol.io/editor-vue` requires Vue 3. In a Vue 2 app, use the core package from
`mounted()` / `beforeDestroy()` with a `ref`-provided container id.

## No build step (CDN)

Only when the page has no bundler — prefer the npm package everywhere else. Both
loaders expose a global; there is no npm involvement and no version pinning, so
you are always on the latest published editor.

```html
<!-- Landing Page Editor -->
<div id="lpe" style="position: absolute; width: 100%; height: 100%"></div>
<script src="https://v1.page-assets.topol.io/topol-lpe.js"></script>
<script>
  const LPE = LandingPageEditor({
    config: { authorize: { apiKey: 'YOUR_API_KEY', userId: 'user-123' } },
    onSave(json, html) {},
  });
  LPE.render('#lpe');
</script>
```

The npm package does exactly this internally — it injects the loader script and
then calls the global factory (`window.LandingPageEditor(options)` for the LPE,
`window.TopolPlugin.init(options)` for the Email Editor).
