# Svelte — `@topol.io/editor-svelte`

Peer dep: `svelte` `^4.2.8`. **Email Editor only** — there is no Landing Page
Editor component for Svelte. For an LPE in a Svelte app, drive the core package
directly (see the end of this file).

```bash
npm install @topol.io/editor-svelte@<resolved 1.x alpha version>
```

## Email Editor

```svelte
<script lang="ts">
  import { EmailEditor, type ISvelteOptions, type IEmailCallbacks } from '@topol.io/editor-svelte';

  const options: ISvelteOptions = {
    authorize: {
      apiKey: import.meta.env.VITE_TOPOL_API_KEY,
      userId: currentUser.id,   // integer or [A-Za-z0-9_-]+
    },
  };

  // Handlers whose result the editor must await go in the `callbacks` prop.
  const callbacks: IEmailCallbacks = {
    async onSave({ json, html, mutations, syncedSections }) {
      // TODO: persist the template — application decision. A rejection shows a save error.
    },
  };

  let component: EmailEditor;
  const save = () => component.getEditor()?.save();
</script>

<button on:click={save}>Save</button>
<div style="height: 100vh">
  <EmailEditor
    bind:this={component}
    {options}
    {callbacks}
    on:onEditorError={(e) => console.error(e.detail.source, e.detail.message)}
    on:ready={(e) => { const editor = e.detail; }}
  />
</div>
```

- `options` is the config (`IEmailConfig`, exported as `ISvelteOptions`).
- Events are dispatched with Svelte 4's dispatcher: `on:onSave`,
  `on:onEditorError`, … with the exact core object payload in `event.detail`
  (language events: `event.detail.code`). They are notifications: put `onSave` /
  `onSaveAndClose` that persist asynchronously, and `getNonce`, in the `callbacks`
  prop. If both exist, the callback runs first and its result goes back to the
  editor.
- `bind:this` + `getEditor()` returns the instance (or `null`); `on:ready`
  delivers it once the iframe has rendered. `onInit` means authenticated.
- The component takes no attributes; it renders `width: 100%; height: 100vh`, so
  size its parent element.
- Props: `options`, `callbacks`, `stage`, `loaderUrl` (leave the last two unset in
  production).
- `ITopolOptions` exported from this package is the core factory type
  `{ config, ...callbacks }`, not the component options — use `ISvelteOptions`.
- `TopolEditor` is an alias of `EmailEditor`; prefer `EmailEditor`.

## Landing Page Editor in Svelte

No component exists — install the core package alongside the wrapper (the same
version the wrapper depends on) and drive the instance yourself:

```svelte
<script lang="ts">
  import { onDestroy, onMount } from 'svelte';
  import { LandingPageEditor, type ILandingPageEditorInstance } from '@topol.io/editor';

  let instance: ILandingPageEditorInstance | null = null;

  onMount(async () => {
    instance = await LandingPageEditor.init({
      config: {
        authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, userId: currentUser.id },
      },
      async onSave({ json, html }) {/* TODO: persist */},
    });
    await instance.render('#lpe');
  });

  onDestroy(() => instance?.destroy());
</script>

<div id="lpe" style="position: absolute; width: 100%; height: 100vh"></div>
```

See `vanilla.md` for the full core API.

## SvelteKit

Init must not run on the server. The component initializes in `onMount`, so it is
SvelteKit-safe as written — keep the editor page client-rendered and do not call
the core `init()` from module scope.
