# @topol.io/editor-react 0.x → 1.x

## 1. The default export is gone

v0.x pointed the package types at the component file, so this worked:

```tsx
import TopolEditor from '@topol.io/editor-react';
```

v1 exports through `entry.ts` with **named exports only**. Both of these work:

```tsx
import { EmailEditor } from '@topol.io/editor-react';   // preferred
import { TopolEditor } from '@topol.io/editor-react';   // deprecated alias, warns on mount
```

Migrate to `EmailEditor` and rename the JSX tags with it. If the app aliases the
import (`import Editor from …`), keep the local name:

```tsx
import { EmailEditor as Editor } from '@topol.io/editor-react';
```

## 2. Options type

```diff
-import TopolEditor, { IReactTopolOptions } from '@topol.io/editor-react';
+import { EmailEditor, type IReactEmailOptions } from '@topol.io/editor-react';

-const options: IReactTopolOptions = { authorize: { apiKey, userId } };
+const options: IReactEmailOptions = { authorize: { apiKey, userId } };
```

`IReactTopolOptions` still exists as a deprecated alias. Do **not** switch to
`ITopolOptions` from this package — in v1 that is the core options type and
requires `id` and `callbacks`.

## 3. Container id changed

The component now renders into `#topol-email-editor-id` instead of `#editor`.
Grep the app for styling or DOM access against the old id and report every hit:

```bash
grep -rn "#editor\b\|getElementById('editor')" . \
  --include='*.css' --include='*.scss' --include='*.ts' --include='*.tsx' \
  --exclude-dir=node_modules
```

## 4. Callback props

Existing handlers keep working. Widened/extended props:

```tsx
onSave?(json, html, mutations, syncedSections)
onSaveAndClose?(json, html, mutations, syncedSections)
onError?(type, message, responseBody?)
onTestSend?(email: string | string[], json, html)   // was string
```

Any `onTestSend` handler that assumed a single address needs a decision from the
user — ask rather than guessing.

New optional props (ignore unless the app uses multilingual templates):
`onBannerClick`, `onLanguageCreated`, `onLanguageDeleted`, `onLanguageSelected`,
`onPrimaryLanguageChanged`, `onGetMutations`.

## Known issue: `tsc` cannot find the package types (alpha.4, alpha.5)

The published alphas declare `"types": "./dist/types/src/entry.d.ts"` but ship
the file at `./dist/types/entry.d.ts`, so after the bump `tsc --noEmit` fails
with:

```
error TS7016: Could not find a declaration file for module '@topol.io/editor-react'.
```

This is an upstream packaging bug, not a migration mistake. Until a fixed alpha
is published, point type resolution at the real file with a `paths` mapping in
the app's `tsconfig.json` (runtime resolution via `exports` is unaffected; never
edit `node_modules`):

```jsonc
{
  "compilerOptions": {
    "paths": {
      "@topol.io/editor-react": [
        "./node_modules/@topol.io/editor-react/dist/types/entry.d.ts"
      ]
    }
  }
}
```

Leave a comment on the mapping and tell the user to remove it once the package
fixes the path.

## Completed example

```tsx
import { EmailEditor, type IReactEmailOptions } from '@topol.io/editor-react';

const options: IReactEmailOptions = {
  authorize: { apiKey: process.env.TOPOL_API_KEY!, userId: 'user-123' },
};

export function EditorScreen() {
  return (
    <EmailEditor
      options={options}
      onSave={(json, html) => save(json, html)}
      onError={(type, message, responseBody) => report(type, message, responseBody)}
    />
  );
}
```

## Adding the Landing Page Editor (optional, new in v1)

```tsx
import { useRef } from 'react';
import { LandingPageEditor, type LandingPageEditorRef, type IReactLandingPageOptions } from '@topol.io/editor-react';

const options: IReactLandingPageOptions = { authorize: { apiKey, userId } };

const ref = useRef<LandingPageEditorRef>(null);
// ref.current?.load(templateJson); ref.current?.save();

<LandingPageEditor ref={ref} options={options} onSave={(json, html) => save(json, html)} />
```

`IReactLandingPageOptions` is the editor **config** object (the wrapper supplies
`config` and the callbacks itself). It renders into `#topol-landing-page-editor-id`.
