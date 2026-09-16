# @topol.io/editor (core) 0.x → 1.x

The core package is the gentlest upgrade: **nothing breaks**. The default export
is still `TopolPlugin`, and `TopolPlugin.init(options, { stage })` is unchanged.

## Recommended rename

```diff
-import TopolPlugin from '@topol.io/editor';
+import { EmailEditor } from '@topol.io/editor';

-await TopolPlugin.init({ id: '#editor', authorize: { apiKey, userId }, callbacks: { … } });
+await EmailEditor.init({ id: '#editor', authorize: { apiKey, userId }, callbacks: { … } });
```

`TopolPlugin` remains exported (default and named) as a deprecated alias of
`EmailEditor`. With the core package you control the container id yourself via
`options.id`, so no DOM/CSS change is forced on you.

## Callbacks to review

```ts
callbacks: {
  onSave(json, html, mutations, syncedSections) {},   // 2 new args
  onError(type, message, responseBody) {},            // new 3rd arg
  onTestSend(email /* string | string[] */, json, html) {},
}
```

Old handlers still compile — the extra parameters are additive. `onTestSend`'s
widened `email` is the one to check by hand.

## Types

All the types the app already used are still exported from the package root
(`ITopolOptions`, `INotification`, `ISavedBlock`, `IMergeTag`, `IMergeTagGroup`,
`ITheme`, `IFont`, `IAPI`, `IStage`, …), plus new ones listed in
`breaking-changes.md`.

## Adding the Landing Page Editor

See `landing-page-editor.md`.
