# Svelte — `@topol.io/editor-svelte`

Peer dep: `svelte` `^4.2.8`. **Email Editor only** — there is no Landing Page
Editor component for Svelte in v1. For an LPE in a Svelte app, drive the core
package directly (see the end of this file).

```bash
npm install @topol.io/editor-svelte@<resolved-v1-version>
```

## Email Editor

The component is named `TopolEditor` here — unlike React/Vue, this is **not**
deprecated in Svelte and there is no `EmailEditor` alias.

```svelte
<script lang="ts">
  import { TopolEditor, type ITopolOptions } from '@topol.io/editor-svelte';

  const options: ITopolOptions = {
    authorize: {
      apiKey: import.meta.env.VITE_TOPOL_API_KEY,
      userId: currentUser.id,
    },
  };

  const handleSave = (event: CustomEvent) => {
    const { json, html, mutations, syncedSections } = event.detail;
    // TODO: persist the template — application decision
  };
</script>

<TopolEditor {options} on:onSave={handleSave} />
```

- Svelte 4 event dispatcher: listen with `on:onSave`, `on:onError`, … and read
  the payload from `event.detail`.
- Payloads are single objects, as in Vue: `onSave`/`onSaveAndClose` →
  `{ json, html, mutations, syncedSections }`, `onTestSend` → `{ email, json, html }`,
  `onError` → `{ type, message, responseBody }`. Note `onLanguageSelected` and
  `onGetMutations` dispatch their value directly, not wrapped.
- `options` is `Omit<ITopolOptions, 'id' | 'callbacks'>` (exported as
  `ISvelteOptions`, re-exported as `ITopolOptions`).
- Container id is **`topol-editor-id`** (not `topol-email-editor-id` — the Svelte
  wrapper kept the 0.x id), rendered at `position: absolute; width: 100%; height: 100vh`.
- `stage` prop defaults to `'production'`.
- Inits in `onMount`, calls `TopolPlugin.destroy()` in `onDestroy`.

The package also re-exports `TopolPlugin` for the imperative methods:

```ts
import { TopolPlugin } from '@topol.io/editor-svelte';
TopolPlugin.save();
```

## Landing Page Editor in Svelte

No component exists — install the core package alongside the wrapper (same exact
version) and drive the instance yourself:

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
      onSave(json, html) {/* TODO: persist */},
    });
    instance.render('#lpe');
  });

  onDestroy(() => instance?.destroy());
</script>

<div id="lpe" style="position: absolute; width: 100%; height: 100vh"></div>
```

See `vanilla.md` for the full core API.

## SvelteKit

Init must not run on the server. `onMount` already guarantees that, so the
component is SvelteKit-safe as written — but do not move the import to a
top-level module that runs during SSR side effects, and keep the editor page
client-rendered.
