# @topol.io/editor-vue 0.x → 1.x

## 1. Component rename

```diff
-import { TopolEditor } from '@topol.io/editor-vue';
+import { EmailEditor } from '@topol.io/editor-vue';
```

```diff
-<TopolEditor :options="options" @on-save="handleSave" />
+<EmailEditor :options="options" @on-save="handleSave" />
```

`TopolEditor` still exists as a deprecated alias (console warning on mount), so
this rename can be staged, but do it in the same pass when practical.

## 2. Options type — the one that breaks builds

v0.x exported `ITopolOptions` as an alias of the Vue-specific options
(`Omit<ITopolOptions, 'id' | 'callbacks'>`). v1 re-exports the **core**
`ITopolOptions`, which still requires `id` and `callbacks`, so this now errors:

```diff
-import { TopolEditor, type ITopolOptions } from '@topol.io/editor-vue';
-const options: ITopolOptions = { authorize: { apiKey, userId } };
+import { EmailEditor, type IVueEmailOptions } from '@topol.io/editor-vue';
+const options: IVueEmailOptions = { authorize: { apiKey, userId } };
```

`IVueOptions` is also still exported and equals `IVueEmailOptions`.

## 3. Container id changed

`#topol-editor-id` → `#topol-email-editor-id`. Grep and report:

```bash
grep -rn "topol-editor-id" . \
  --include='*.css' --include='*.scss' --include='*.vue' --include='*.ts' \
  --exclude-dir=node_modules
```

## 4. Payload types moved

`ISaveData`, `ISendTestData`, and `IErrorData` now come from the core package and
are re-exported from `@topol.io/editor-vue`. Import them from the package root,
not from the component file:

```diff
-import type { ISaveData } from '@topol.io/editor-vue/dist/TopolEditor.vue';
+import type { ISaveData } from '@topol.io/editor-vue';
```

`ISaveData` now also carries `mutations` and `syncedSections`; `ISendTestData.email`
is `string | string[]` (see `breaking-changes.md`).

## 5. New events (optional)

`@on-language-created`, `@on-language-deleted`, `@on-language-selected`,
`@on-primary-language-changed`, `@on-get-mutations` — each emits an object payload.

## Completed example

```vue
<script setup lang="ts">
import { EmailEditor, type IVueEmailOptions, type ISaveData } from '@topol.io/editor-vue';

const options: IVueEmailOptions = {
  authorize: { apiKey: import.meta.env.VITE_TOPOL_API_KEY, userId: 'user-123' },
};

function handleSave({ json, html }: ISaveData) {
  // …
}
</script>

<template>
  <EmailEditor :options="options" @on-save="handleSave" />
</template>
```

## Adding the Landing Page Editor (optional, new in v1)

```vue
<script setup lang="ts">
import { LandingPageEditor, type IVueLandingPageOptions } from '@topol.io/editor-vue';

const options: IVueLandingPageOptions = { authorize: { apiKey, userId } };
</script>

<template>
  <LandingPageEditor :options="options" @on-save="handleSave" />
</template>
```

Renders into `#topol-landing-page-editor-id`.
