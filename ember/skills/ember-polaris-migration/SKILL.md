---
name: ember-polaris-migration
description: Migrating Ember Octane apps toward Polaris — single-file components (.gjs/.gts), the <template> tag, strict-mode templates, route templates as components, and the mental model shifts. Use when introducing template imports to a codebase, when starting a new Polaris-first project, or when you see <template> blocks and need to understand them.
type: reference
---

# Ember Polaris Migration

**Polaris** is the next Ember edition. It's an *additive* set of changes on top of Octane — you don't unlearn Octane to adopt Polaris. The changes are mostly about *how templates are authored* and *how names resolve*.

The big shifts:

1. **Single-file components** with `.gjs` / `.gts` and a `<template>` block.
2. **Strict-mode templates** — names come from JS imports, not the resolver.
3. **Route templates as components** (RFC #1099) — eliminates controllers as the data home for route URLs.
4. **First-class JS in templates** — no helper wrappers around plain functions.

Polaris depends on **Embroider** (`@embroider/core` etc.). Classic builds can't run it.

## What `.gjs` / `.gts` look like

```ts
// app/components/post-card.gts
import Component from '@glimmer/component';
import { on } from '@ember/modifier';
import { fn } from '@ember/helper';
import FormatDate from 'my-app/helpers/format-date';
import LikeButton from 'my-app/components/like-button';
import type PostModel from 'my-app/models/post';

export interface PostCardSignature {
  Args: { post: PostModel };
  Element: HTMLElement;
}

export default class PostCard extends Component<PostCardSignature> {
  share = () => navigator.share?.({ title: this.args.post.title });

  <template>
    <article ...attributes>
      <h2>{{@post.title}}</h2>
      <time>{{FormatDate @post.publishedAt format="long"}}</time>

      <LikeButton @postId={{@post.id}} />

      <button type="button" {{on "click" this.share}}>
        Share
      </button>
    </article>
  </template>
}
```

Notes:
- `<template>` is a magic block — *not* a real DOM element. It compiles to the component's template.
- Things used in the template (`on`, `LikeButton`, `FormatDate`) must be **imported**.
- Built-ins like `if`, `each`, `let` are still globally available.

## Strict mode = imported scope

In classic mode, `<UserCard>` is resolved by the file system: `app/components/user-card.{ts,hbs}`. In strict mode, *every name in a template must be in scope* — either:

- A class field (`this.share`).
- An arg (`@post`).
- An import (`LikeButton`, `FormatDate`).
- A built-in keyword (`if`, `each`, `let`, `unless`, `with`, `each-in`, `yield`, `outlet`, `mount`, `component`, `helper`, `modifier`, `array`, `hash`, `concat`, `debugger`, `log`).

This kills "where is this component defined?" mysteries. Click-through-to-definition works.

## Template-only components

```ts
// app/components/badge.gts
import type { TOC } from '@ember/component/template-only';

export interface BadgeSignature {
  Args: { label: string };
  Element: HTMLSpanElement;
}

const Badge: TOC<BadgeSignature> = <template>
  <span class="badge" ...attributes>{{@label}}</span>
</template>;

export default Badge;
```

`TOC<T>` typing tells Glint to check the template against the signature.

## Helpers and modifiers as plain functions

Helpers are just functions you import:

```ts
// app/helpers/format-date.ts
export default function formatDate(date: Date, options: Intl.DateTimeFormatOptions = {}) {
  return new Intl.DateTimeFormat('en-US', options).format(date);
}
```

```ts
// usage in a .gts file
import FormatDate from 'my-app/helpers/format-date';

<template>{{FormatDate @publishedAt month="long"}}</template>
```

No `helper(...)` wrapper, no `class extends Helper`. If the helper needs DI, you can still extend `Helper`, but most helpers don't.

Modifiers can also be plain functions via `ember-modifier`:

```ts
import { modifier } from 'ember-modifier';

const autoFocus = modifier((el: HTMLElement) => el.focus());
export default autoFocus;
```

```ts
import autoFocus from 'my-app/modifiers/auto-focus';

<template>
  <input {{autoFocus}} />
</template>
```

## Route templates as components (RFC #1099)

Currently, `app/templates/posts/show.hbs` is a special template. Polaris's route-as-component proposal makes it a regular component you export from the route file:

```ts
// app/routes/posts/show.gts (proposed Polaris pattern)
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class PostsShowRoute extends Route<Promise<Post>> {
  @service declare store: Store;
  async model({ post_id }: { post_id: string }) {
    return this.store.findRecord('post', post_id);
  }

  <template>
    <article>
      <h1>{{@model.title}}</h1>
      <p>{{@model.body}}</p>
      {{outlet}}
    </article>
  </template>
}
```

Status: this RFC is in flight — check the latest Ember release notes. Until it lands, keep templates separate at `app/templates/...` and lift complex logic into components.

## Migration: side-by-side, file by file

You don't migrate everything at once. The two modes coexist:

1. **Bootstrap Embroider** if you haven't already (`@embroider/core` + `@embroider/compat` + `@embroider/webpack`).
2. **Install template-imports support**:
   ```bash
   ember install @glimmer/component
   ember install ember-template-imports
   ember install ember-modifier  # most teams already have this
   ```
3. **Add the Glint environment**:
   ```jsonc
   // tsconfig.json
   "glint": { "environment": ["ember-loose", "ember-template-imports"] }
   ```
4. **New components**: write in `.gts`. Don't touch existing `.hbs`/`.ts` pairs.
5. **Migrate old components opportunistically** when you'd touch them anyway.

A reasonable migration sequence per component:

1. Rename `foo.ts` + `foo.hbs` → `foo.gts`.
2. Add `<template>...</template>` containing the old `.hbs` body.
3. Find every helper/component/modifier referenced — add an `import` for each.
4. Run `glint`. Fix the resulting errors.
5. Delete the old `.hbs`.

## Mental model shifts

| Before (Octane) | After (Polaris) |
|---|---|
| File path = component name | Imports = component name |
| Helpers wrapped with `helper(...)` | Helpers are plain functions |
| `register-helper`, classic helper class | Most helpers don't need a class |
| Implicit globals (`{{format-date}}`) | Explicit imports |
| Template registry maintenance | Glint sees imports directly |
| `app/templates/foo.hbs` for routes | (Future) component-shaped routes |
| Two files (`.ts` + `.hbs`) | One file (`.gts`) |

## What does NOT change in Polaris

- `@tracked`, `@action`, `@service`, `@cached` — same.
- DI / owner / registry — same.
- Router, route hooks, model — same (until RFC #1099 lands).
- Ember Data — same.
- Testing (`@ember/test-helpers`, QUnit) — same; tests can render `<template>`-style components directly.

## Tips and traps

- **`type="button"` is not optional** in templates. The lint rule (`require-button-type`) is on by default and Polaris-era code keeps it.
- **`{{this.x}}` vs `{{x}}`**: in strict mode, `{{x}}` is a name lookup that must resolve to an import or built-in. Always use `{{this.x}}` for class fields.
- **Dynamic component invocation** still uses `(component ...)` and `<this.dynamicCmp />`. The curried component reference is just a JS value now.
- **Yielding components**: yield the component reference — `{{yield (component LikeButton @postId=@id)}}` and the parent calls `<x.LikeButton />`.
- **`hbs` template literal in tests**: in template-imports mode, you can render components directly in tests instead of via `hbs\`...\``:
  ```ts
  // .gts test file
  await render(<template><PostCard @post={{this.post}} /></template>);
  ```

## Common questions

**Q: Can I mix `.hbs` + `.gts` in the same app?**
Yes. That's the supported migration path. Use `ember-template-imports` together with the classic resolver.

**Q: Do I have to use Embroider?**
For `.gjs`/`.gts`, yes. For mixed apps, `@embroider/compat` + classic `ember-cli` works during the transition.

**Q: Does `<template>` show up in the DOM?**
No. It compiles away. The first real element in your template is the root element.

**Q: How does `...attributes` interact with template-only components?**
Same as before — declare an `Element` in the signature, place `...attributes` on the root element.

## Verification when introducing Polaris

- [ ] Embroider is in the build pipeline.
- [ ] `ember-template-imports` is installed and Glint includes the `ember-template-imports` environment.
- [ ] New components are `.gts` with explicit imports.
- [ ] `template-lint` is upgraded to a version that supports `.gjs`/`.gts`.
- [ ] Tests render via `<template>` (where convenient) for cleaner setup.
- [ ] No `.hbs`/`.ts` pairs are being added; all new code is single-file.

## See also

- Official: [emberjs/rfcs#779](https://github.com/emberjs/rfcs/pull/779) (template tag), [emberjs/rfcs#496](https://github.com/emberjs/rfcs/pull/496), [emberjs/rfcs#1099](https://github.com/emberjs/rfcs/pull/1099) (route templates as components).
- [typed-ember/glint](https://github.com/typed-ember/glint) — the type-checker.
- `ember-typescript-and-glint` — type-checking templates.
- `ember-components-and-templates` — what `<template>` is replacing.
- `ember-octane-fundamentals` — the foundation Polaris builds on.
