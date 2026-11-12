# React — `@topol.io/editor-react`

Peer deps: `react` / `react-dom` `^18 || ^19`. Ships both editors as components.

```bash
npm install @topol.io/editor-react@<resolved 1.x version>
```

## Email Editor

```tsx
import { useRef } from 'react';
import { EmailEditor, type IEmailEditorRef, type IReactEmailOptions } from '@topol.io/editor-react';

const options: IReactEmailOptions = {
  authorize: {
    apiKey: import.meta.env.VITE_TOPOL_API_KEY,   // or process.env.NEXT_PUBLIC_TOPOL_API_KEY
    entityId: currentUser.id,                     // integer or [A-Za-z0-9_-]+
  },
};

export default function EmailEditorPage() {
  const ref = useRef<IEmailEditorRef>(null);

  return (
    <>
      <button onClick={() => ref.current?.getEditor()?.save()}>Save</button>
      <EmailEditor
        ref={ref}
        options={options}
        onSave={async ({ json, html, mutations, syncedSections }) => {
          // TODO: persist the template — application decision. Return/await the
          // promise: the editor waits for it and shows a rejection as a save error.
        }}
        onEditorError={({ source, message, detail }) => console.error(source, message, detail)}
        onReady={(editor) => {/* iframe rendered; instance available */}}
        style={{ height: '100vh' }}
      />
    </>
  );
}
```

- `options` is the config (`IEmailConfig`); callbacks are separate props with
  the core's **object** payloads and return types.
- `ref.current?.getEditor()` returns the instance (or `null` before it exists);
  every method returns a promise. `onReady` fires once the iframe has rendered;
  `onInit` fires once the editor has authenticated.
- The component renders a plain `div` (`width: 100%; height: 100vh` by default,
  no id). Size it with `style` / `className`.
- Extra props: `stage`, `loaderUrl` (leave unset in production), `getNonce` for
  nonce authentication, `onCloudSave` / `onCloudTemplateChange` for Cloud.

## Landing Page Editor

Same shape as `EmailEditor`.

```tsx
import { useRef } from 'react';
import {
  LandingPageEditor,
  type ILandingPageEditorRef,
  type IReactLandingPageOptions,
} from '@topol.io/editor-react';

const options: IReactLandingPageOptions = {
  authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, entityId: currentUser.id },
};

export default function LandingPagePage() {
  const editorRef = useRef<ILandingPageEditorRef>(null);

  return (
    <>
      <button onClick={() => editorRef.current?.save()}>Save</button>
      <LandingPageEditor
        ref={editorRef}
        options={options}
        onInit={() => {
          // Editor ready: load the first template here, not in onLoaded.
          void editorRef.current?.load(templateJson);
        }}
        onSave={async ({ json, html }) => {/* TODO: persist */}}
        onEditorError={({ source, message }) => console.error(source, message)}
        style={{ height: '100vh' }}
      />
    </>
  );
}
```

- `options` here is the LPE **config** object only (`ILandingPageConfig`);
  the wrapper wraps it as `{ config, ...callbacks }` and calls `render()` itself.
- The ref exposes `load(template)` and `save()`, both returning promises, and
  `getEditor()` for every other instance method (`null` before the editor has
  started). For `destroy`, unmount the component.
- The container is a plain `div` (`width: 100%; height: 100vh`, no id); size it
  with `style` / `className`. Several editors can share a page.
- Extra props: `onReady(editor)`, `loaderUrl`, `getNonce`. `stage` is accepted
  and ignored.
- Only the callbacks you pass are forwarded; without `onSaveAndClose` /
  `onEditorClose` the runtime falls back to `onSave` and closes itself.
- A failed `init()` / `render()` is reported as `onEditorError` with
  `source: 'integration'`.

## Options are read on mount

Both components initialize once on mount and tear down on unmount, including
while init is still pending. Configuration and **which** callback props are
present are captured on mount; changing them afterwards does not re-initialize
the editor. Callback functions are read fresh on every call, so handlers can
close over current state safely.

To change configuration at runtime, call
`ref.current?.getEditor()?.updateOptions(partial)`, or remount with a new React
`key`. Define `options` outside the component or in a `useMemo`.

`StrictMode` double-mounting is handled internally (one instance, one iframe);
do not add your own init guard. Each component owns its own instance, so several
editors per page work, but editors of one kind must share one loader URL.

## Next.js / SSR

The editor needs a browser; `EmailEditor.init` rejects on the server. Mark the
page or component as a client component:

```tsx
'use client';
import { EmailEditor } from '@topol.io/editor-react';
```

If you hit hydration or `window is undefined` errors from a shared layout, import
it lazily instead:

```tsx
const EmailEditor = dynamic(
  () => import('@topol.io/editor-react').then((m) => m.EmailEditor),
  { ssr: false },
);
```

Use `NEXT_PUBLIC_TOPOL_API_KEY` — a non-`NEXT_PUBLIC_` variable is undefined in
the browser and the editor will fail to authorize. In the Pages Router, the same
`dynamic(..., { ssr: false })` import is the standard approach.

## Names that no longer exist

There is no default export and no `TopolEditor` component; import
`{ EmailEditor }`. The core factories are re-exported as `CoreEmailEditor` /
`CoreLandingPageEditor` (not `TopolPlugin`). `LandingPageEditorRef` /
`LandingPageEditorProps` are now `ILandingPageEditorRef` /
`ILandingPageEditorProps`.
