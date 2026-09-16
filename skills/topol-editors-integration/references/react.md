# React — `@topol.io/editor-react`

Peer deps: `react` / `react-dom` `^18 || ^19`. Ships both editors as components.

```bash
npm install @topol.io/editor-react@<resolved-v1-version>
```

## Email Editor

```tsx
import { EmailEditor, type IReactEmailOptions } from '@topol.io/editor-react';

const options: IReactEmailOptions = {
  authorize: {
    apiKey: import.meta.env.VITE_TOPOL_API_KEY,   // or process.env.NEXT_PUBLIC_TOPOL_API_KEY
    userId: currentUser.id,
  },
};

export default function EmailEditorPage() {
  return (
    <EmailEditor
      options={options}
      onSave={(json, html, mutations, syncedSections) => {
        // TODO: persist the template — application decision
      }}
      onError={(type, message, responseBody) => console.error(type, message, responseBody)}
    />
  );
}
```

- `options` is `Omit<ITopolOptions, 'id' | 'callbacks'>` — the component owns the
  container id and turns your props into callbacks.
- The component renders its own `<div id="topol-email-editor-id">` with
  `position: absolute; width: 100%; height: 100vh`. Mount it on a full-page route.
- Callbacks are **positional props**, matching the core signatures.

## Landing Page Editor

```tsx
import { useRef } from 'react';
import {
  LandingPageEditor,
  type LandingPageEditorRef,
  type IReactLandingPageOptions,
} from '@topol.io/editor-react';

const options: IReactLandingPageOptions = {
  authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, userId: currentUser.id },
};

export default function LandingPagePage() {
  const editorRef = useRef<LandingPageEditorRef>(null);

  return (
    <>
      <button onClick={() => editorRef.current?.load(templateJson)}>Load</button>
      <button onClick={() => editorRef.current?.save()}>Save</button>
      <LandingPageEditor
        ref={editorRef}
        options={options}
        onSave={(json, html) => {/* TODO: persist */}}
      />
    </>
  );
}
```

- `options` here is the LPE **config** object only (`ILandingPageOptions['config']`);
  the wrapper wraps it as `{ config, ...callbacks }` and calls `render()` itself.
- The ref exposes `load(template)` and `save()` — nothing else. For `destroy`,
  unmount the component.
- Container id is `topol-landing-page-editor-id`.
- `onSave` takes **two** args here, not four.

## Both editors on one screen

Each component mounts a fixed, absolutely-positioned container, so rendering both
at once overlaps them. Render one at a time (tab/route switch), not side by side.

## Options are read once

Both components initialize in a `useEffect` with an empty dependency array and
tear down on unmount — changing `options` afterwards does **not** re-initialize
the editor. Callback props are always read fresh (they go through a ref), so
handlers can close over current state safely. To change configuration at runtime,
call `CoreEmailEditor.updateOptions(partial)` instead of mutating the prop:

```tsx
import { CoreEmailEditor } from '@topol.io/editor-react';
CoreEmailEditor.updateOptions({ light: true });
```

Define `options` outside the component or in a `useMemo` — a fresh object each
render is harmless but misleading.

`StrictMode` double-mounting is already guarded internally; do not add your own
init guard.

## Next.js / SSR

The editor loads a browser script and touches `window` on init, so it must never
run on the server.

```tsx
'use client';
import { EmailEditor } from '@topol.io/editor-react';
```

`'use client'` at the top of the page/component is enough in the App Router. If
you hit hydration or `window is undefined` errors from a shared layout, import it
lazily instead:

```tsx
const EmailEditor = dynamic(
  () => import('@topol.io/editor-react').then((m) => m.EmailEditor),
  { ssr: false },
);
```

Use `NEXT_PUBLIC_TOPOL_API_KEY` — a non-`NEXT_PUBLIC_` variable is undefined in
the browser and the editor will fail to authorize. In the Pages Router, the same
`dynamic(..., { ssr: false })` import is the standard approach.

## Deprecated

`TopolEditor` is the 0.x component name, kept as an alias that logs a deprecation
warning on mount. Use `EmailEditor` in new code.
