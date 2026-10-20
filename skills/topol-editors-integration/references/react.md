# React — `@topol.io/editor-react`

Peer deps: `react` / `react-dom` `^18 || ^19`. Ships both editors as components.

```bash
npm install @topol.io/editor-react@<resolved 1.x alpha version>
```

## Email Editor

```tsx
import { useRef } from 'react';
import { EmailEditor, type IEmailEditorRef, type IReactEmailOptions } from '@topol.io/editor-react';

const options: IReactEmailOptions = {
  authorize: {
    apiKey: import.meta.env.VITE_TOPOL_API_KEY,   // or process.env.NEXT_PUBLIC_TOPOL_API_KEY
    userId: currentUser.id,                       // integer or [A-Za-z0-9_-]+
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
- Extra props: `stage`, `loaderUrl` (leave unset in production), `getNonce`,
  `onCloudSave`, `onCloudTemplateChange` for nonce authentication / Cloud.

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
        onSave={async ({ json, html }) => {/* TODO: persist */}}
        onEditorError={({ source, message }) => console.error(source, message)}
      />
    </>
  );
}
```

- `options` here is the LPE **config** object only (`ILandingPageOptions['config']`);
  the wrapper wraps it as `{ config, ...callbacks }` and calls `render()` itself.
- The ref exposes `load(template)` and `save()`. For `destroy`, unmount the
  component.
- The container is `#topol-landing-page-editor-id`, `position: absolute;
  width: 100%; height: 100vh` — mount it on a full-page route.
- Only the callbacks you pass are forwarded; without `onSaveAndClose` /
  `onEditorClose` the runtime falls back to `onSave` and closes itself.

## Options are read on mount

Both components initialize once on mount and tear down on unmount. Configuration
and **which** callback props are present are captured on mount; changing them
afterwards does not re-initialize the editor. Callback functions are read fresh
on every call, so handlers can close over current state safely.

To change Email Editor configuration at runtime, call
`ref.current?.getEditor()?.updateOptions(partial)`, or remount with a new React
`key`. Define `options` outside the component or in a `useMemo`.

`StrictMode` double-mounting is handled internally (one instance, one iframe);
do not add your own init guard. Each component owns its own instance, so several
Email Editors per page work, but they must share one loader URL.

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

## Deprecated

`TopolEditor` is a named, deprecated alias of `EmailEditor` with the same props;
it logs a warning on mount. There is no default export. Use `EmailEditor`.
