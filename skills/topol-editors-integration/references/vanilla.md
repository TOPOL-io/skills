# Plain JS/TS — `@topol.io/editor`

The core package. Use it when there is no supported framework wrapper (vanilla
JS/TS, Angular, Vue 2, Svelte 4, Svelte for the LPE) or when you need imperative
control.

```bash
npm install @topol.io/editor@<resolved 1.x version>
```

ESM and CJS builds are both published; types ship with the package. Both editors
are named exports; there is no default export.

## Email Editor

```html
<div id="editor" style="width: 100%; height: 100vh"></div>
```

```ts
import { EmailEditor, type IEmailConfig } from '@topol.io/editor';

const config: IEmailConfig = {
  authorize: { apiKey: TOPOL_API_KEY, entityId: currentUser.id },   // entityId: integer or [A-Za-z0-9_-]+
};

const editor = await EmailEditor.init({
  config,
  async onSave({ json, html, mutations, syncedSections }) {
    // TODO: persist the template — application decision. A rejection shows a save error.
  },
  onEditorError({ source, message, detail }) {
    console.error(source, message, detail);
  },
  onInit() {
    // editor authenticated; methods that need it are safe now
  },
});
await editor.render('#editor');   // selector or HTMLElement; must exist and have a height

// later, from your own UI:
await editor.save();
// on teardown (not right after render):
await editor.destroy();
```

- `init()` returns a promise — `await` it (or `.catch()`). It rejects without
  `config.authorize`, with an invalid `entityId`, outside a browser, or for loader
  conflicts.
- `init()` resolves to an **instance**; nothing appears until `render()` runs.
  Each `init()` creates an independent instance, so several editors per page
  work (all on one loader URL).
- Callbacks sit at the top level next to `config`; every callback takes no
  arguments or one object. See `options.md` for the full callback list and the
  instance methods.
- Second argument: `{ stage?, loaderUrl? }`. Leave it out in production code.
- Let `init()` load the editor script. Do not add `<script src=".../loader/build.js">`
  yourself — `init()` rejects a host-inserted loader.

## Landing Page Editor

Instance-based with the same `{ config, ...callbacks }` shape.

```html
<div id="lpe" style="width: 100%; height: 100vh"></div>
```

```ts
import { LandingPageEditor, type ILandingPageEditorInstance } from '@topol.io/editor';

const instance: ILandingPageEditorInstance = await LandingPageEditor.init({
  config: {
    authorize: { apiKey: TOPOL_API_KEY, entityId: currentUser.id },
  },
  async onSave({ json, html }) {/* TODO: persist */},
  onEditorError({ source, message }) { console.error(source, message); },
});

await instance.render('#lpe');      // nothing renders until this runs
await instance.load(templateJson);  // optional: first template (never from onLoaded)
// await instance.save();           → triggers onSave, resolves once it settled
// await instance.destroy();
```

Every instance method returns a promise. The LPE config is validated at runtime:
a wrongly-typed option stops the editor from starting and is reported through
`onEditorError` (`source: 'init'`); unknown keys are ignored. The second
`init()` argument takes `{ loaderUrl? }` (`stage` is ignored).

## Angular

Init from `ngAfterViewInit` (the container must exist in the DOM), keep the
instance on the component, and call `await editor.destroy()` from `ngOnDestroy`.
Keep the container element outside any `*ngIf` that could tear it down while the
editor is live. Nothing else is special — it is the vanilla flow above.
`@topol.io/editor-angular` is a separate 0.x package on Email Editor v3; do not
install it for a 1.x integration.

## Vue 2

`@topol.io/editor-vue` requires Vue 3. In a Vue 2 app, use the core package from
`mounted()` (pass `this.$refs.container` to `render`) and destroy the instance in
`beforeDestroy()`.

## No build step (CDN)

Only when the page has no bundler — prefer the npm package everywhere else. The
loader scripts expose global factories that take the same `{ config, ...callbacks }`
and return the same instance as the package's `init()`, synchronously.

- **Email Editor**:

  ```html
  <div id="editor" style="width: 100%; height: 100vh"></div>
  <script src="https://v4.email-assets.topol.io/loader/build.js"></script>
  <script>
    const editor = TopolEmailEditor({
      config: { authorize: { apiKey: 'YOUR_API_KEY', entityId: 'user-123' } },
      onSave({ json, html }) {},
    });
    editor.render('#editor');
  </script>
  ```

- **Landing Page Editor**:

  ```html
  <div id="lpe" style="width: 100%; height: 100vh"></div>
  <script src="https://v1.page-assets.topol.io/loader/build.js"></script>
  <script>
    const LPE = LandingPageEditor({
      config: { authorize: { apiKey: 'YOUR_API_KEY', entityId: 'user-123' } },
      onSave({ json, html }) {},
    });
    LPE.render('#lpe');
  </script>
  ```

The npm package adds `entityId` validation, loader conflict checks and cleanup,
so use it whenever there is a bundler, and do not mix a script tag and the
package for the same editor on one page. Snippets that call `TopolPlugin.init`
are the older v3 API and do not apply.
