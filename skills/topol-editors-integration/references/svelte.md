# Svelte — `@topol.io/editor-svelte`

Peer dep: `svelte` `^5.0.0`. **Svelte 5 only**, and **Email Editor only** — there
is no Landing Page Editor component for Svelte. For an LPE in a Svelte app, drive
the core package directly (see the end of this file).

A Svelte 4 app cannot load this package. Drive the core `@topol.io/editor` from
`onMount` instead (`vanilla.md`), or upgrade Svelte first; ask the user which.

```bash
npm install @topol.io/editor-svelte@<resolved 1.x version>
```

## Email Editor

```svelte
<script lang="ts">
  import {
    EmailEditor,
    type ISvelteEmailOptions,
    type IEmailEditorInstance,
    type IEmailSavePayload,
  } from '@topol.io/editor-svelte';

  const options: ISvelteEmailOptions = {
    authorize: {
      apiKey: import.meta.env.VITE_TOPOL_API_KEY,
      entityId: currentUser.id,   // integer or [A-Za-z0-9_-]+
    },
  };

  let editor: IEmailEditorInstance | null = null;

  async function onSave({ json, html, mutations, syncedSections }: IEmailSavePayload) {
    // TODO: persist the template — application decision. The editor awaits this
    // promise; a rejection shows a save error.
  }
</script>

<button onclick={() => editor?.save()}>Save</button>
<EmailEditor
  {options}
  {onSave}
  onEditorError={({ source, message, detail }) => console.error(source, message, detail)}
  onReady={(instance) => (editor = instance)}
  style="height: 100vh"
/>
```

- `options` is the config (`IEmailConfig`, exported as `ISvelteEmailOptions`).
- **Every callback is a prop** that receives the core's payload object directly:
  `onSave={...}`, `onEditorError={...}`, `onLanguageCreated={({ code }) => ...}`,
  `getNonce={...}`. A returned promise (`onSave`, `onSaveAndClose`,
  `onEditorClose`, `getNonce`) is awaited by the editor. There are no `on:`
  events and no `callbacks` prop. `IEmailEditorProps` types all props.
- `onReady(editor)` delivers the instance once the iframe has rendered;
  `bind:this={component}` + `component.getEditor()` returns it too (or `null`).
  `onInit` means authenticated.
- The container is `width: 100%; height: 100vh` by default; the `class` and
  `style` props change it.
- Other props: `stage`, `loaderUrl` (leave both unset in production).
- Configuration and which callback props are present are captured on mount; wrap
  the component in a `{#key}` block to change them. A callback that was present
  on mount always calls its latest value.

## Landing Page Editor in Svelte

No component exists — install the core package alongside the wrapper (the same
version the wrapper depends on) and drive the instance yourself:

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { LandingPageEditor, type ILandingPageEditorInstance } from '@topol.io/editor';

  let container: HTMLDivElement;

  onMount(() => {
    let instance: ILandingPageEditorInstance | undefined;
    let disposed = false;

    (async () => {
      const created = await LandingPageEditor.init({
        config: {
          authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, entityId: currentUser.id },
        },
        async onSave({ json, html }) {/* TODO: persist */},
        onEditorError({ source, message }) { console.error(source, message); },
      });
      if (disposed) return created.destroy();   // unmounted while init was pending
      instance = created;
      await instance.render(container);
    })();

    return () => {
      disposed = true;
      void instance?.destroy();
    };
  });
</script>

<div bind:this={container} style="width: 100%; height: 100vh"></div>
```

See `vanilla.md` for the full core API.

## SvelteKit

Init must not run on the server. The component initializes in `onMount`, so it is
SvelteKit-safe as written — do not call the core `init()` from module scope or a
`load` function.

## Names that no longer exist

`TopolEditor`, the `TopolPlugin` re-export, `on:onSave`-style events, the
`callbacks` prop, `on:ready`, `ISvelteOptions` (now `ISvelteEmailOptions`) and
`ISvelteEmailEvents` are gone. The core factory is re-exported as
`CoreEmailEditor`.
