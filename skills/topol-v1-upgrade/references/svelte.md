# @topol.io/editor-svelte 0.x → 1.x

**No API change.** The Svelte wrapper's public surface is identical in 1.x:

```ts
import { TopolEditor, TopolPlugin } from '@topol.io/editor-svelte';
import type { ITopolOptions, INotification, IStage, ISavedBlock } from '@topol.io/editor-svelte';
```

The upgrade is a version bump only. Bump it together with `@topol.io/editor` so
the wrapper and core stay on the same major.

Two things to tell the user:

1. The **core** callback payloads still changed underneath (extra `mutations` /
   `syncedSections` args on save, `responseBody` on `onError`, `onTestSend` email
   may be an array). See `breaking-changes.md`.
2. There is **no Svelte Landing Page Editor component**. A Svelte app that wants
   the landing page editor calls the core API directly — see
   `landing-page-editor.md` for the vanilla pattern, driven from `onMount`.
