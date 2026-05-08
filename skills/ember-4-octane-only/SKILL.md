---
name: ember-4-octane-only
description: What's true (and what's gone) in Ember 4.x — Octane is mandatory, jQuery is removed, Ember.X globals are gone, classic components fail to compile, observers mostly gone, ember-data 4 typing tightened. Use when triaging a 4.x app, when explaining why classic patterns no longer compile, or when reviewing PRs against a 4.x codebase.
type: reference
---

# Ember 4.x — The Octane-Only Era

4.x is where the classic API stopped being a fallback. Code that still ran on 3.28 with deprecation warnings now **fails to compile** on 4.x. This skill enumerates what's gone, what survives, and what's *new* (specific to the 4.x window) that 5.x then reshaped.

## The mental model

Identical to the [`ember-octane-fundamentals`](../ember-octane-fundamentals) skill. If you can write modern Octane (native classes, `@tracked`, `@action`, `@service`, modifiers), you can write 4.x.

**4.x-specific reminder:** the `<template>` tag (`.gjs`/`.gts`) is **not** in 4.x. Stay on classic colocated `.ts` + `.hbs` files (or `.js` + `.hbs`).

## What's gone in 4.x (vs. late 3.x)

| Removed | Replacement |
|---|---|
| `Ember.Component` (the classic component class) | `@glimmer/component` |
| `Ember.Object.extend({...})` | Native classes |
| `Ember.Mixin` | Services or utility modules |
| `Ember.computed` (the global) | Getters with `@tracked` reads (or `import { computed } from '@ember/object'` for the rare cases) |
| `Ember.observer` (the global) | Derived state, modifiers, or explicit action chaining |
| jQuery integration (the `@ember/optional-features` flag and bundled `$`) | Native DOM, `fetch`, custom modifiers |
| `Ember.run` (the global) | `import { run, scheduleOnce } from '@ember/runloop'` |
| `Ember.RSVP` (the global) | `import RSVP from 'rsvp'` (or native promises) |
| `Ember.String.htmlSafe` etc. | `import { htmlSafe } from '@ember/template'` |
| `Ember.copy`, `Ember.merge`, `Ember.assign` | Native `structuredClone`, spread, `Object.assign` |
| `partial` template helper | Inline or extract to component |
| `with` template helper | `let` |
| Classic test API (`moduleForComponent`, `moduleFor`, `moduleForAcceptance`) | `module + setupRenderingTest/setupTest/setupApplicationTest` |
| `(action ...)` template helper | `(fn ...)` and `{{on "event" this.handler}}` |
| `(mut ...)` template helper | Pass callbacks back up |
| `sendAction` | Closure functions / args |
| `ember-cli-typescript` as a separate addon | Built-in TS support (configure `tsconfig.json` directly) |

## What is new in 4.x

| New / matured | What it gives you |
|---|---|
| Official TypeScript support | Without the `ember-cli-typescript` addon — the framework ships its own type definitions. Glint is still maturing, so most 4.x apps run plain `tsc --noEmit` for the JS layer and skip template type-checking. |
| Embroider becomes practical | `@embroider/core` + `@embroider/compat` is the recommended starting point. `optimized` mode is achievable on greenfield 4.x apps. |
| `ember-data` 4 typing | `Model` typing tightened; `attr`/`belongsTo`/`hasMany` declarations need explicit options for clean TS. |
| `@ember/test-waiters` | Canonical replacement for `Ember.Test.registerWaiter`. |
| Better tree-shaking | Embroider compat mode tree-shakes most addons. The 3.x build pipeline didn't. |
| `tracked-built-ins` | `TrackedArray`/`TrackedMap`/`TrackedSet` for reactive collections. Available earlier, but matures into the canonical answer in 4.x. |

## What survives unchanged

These are exactly the same in 4.x as in 3.x-Octane and 5.x:

- Routing API (`Router.map`, route hooks, `RouterService`).
- `@service` injection and the container.
- `@tracked` semantics.
- `@action` semantics.
- `@cached` for memoized getters.
- Component signatures (Args / Element / Blocks).
- Mirage, page-object, test-selectors, intl, simple-auth, concurrency, modifier — all major addons work the same way.

## What's *not yet* in 4.x

These are 5.x+ features. If you see them, the reader is in 5.x territory or aspirational:

- `<template>` tag in `.gjs`/`.gts` files. Not available; you author `.ts` + `.hbs`.
- Strict-mode templates (template imports). Not available.
- Glint's `ember-template-imports` environment. The `ember-loose` environment partially works but with rough edges; many teams disable Glint on 4.x.
- The new ember-data request manager (`@ember-data/request`) as the *default*. Available for opt-in late in 4.x; default in 5.x.
- WarpDrive branding for the next-gen ember-data. (5.x+.)
- Native browser-supported decorators. 4.x still uses TS legacy/experimental decorators.

## Anatomy of a 4.x component

```ts
// app/components/post-card.ts
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';
import { service } from '@ember/service';
import type RouterService from '@ember/routing/router-service';
import type PostModel from 'my-app/models/post';

export interface PostCardSignature {
  Args: {
    post: PostModel;
  };
  Element: HTMLElement;
}

export default class PostCard extends Component<PostCardSignature> {
  @service declare router: RouterService;
  @tracked isExpanded = false;

  @action toggle() {
    this.isExpanded = !this.isExpanded;
  }
}
```

```hbs
{{! app/components/post-card.hbs }}
<article ...attributes>
  <h2>{{@post.title}}</h2>
  {{#if this.isExpanded}}
    <p>{{@post.body}}</p>
  {{/if}}
  <button type="button" {{on "click" this.toggle}}>
    {{if this.isExpanded "Collapse" "Expand"}}
  </button>
</article>
```

This is identical to a modern Ember component minus the `<template>` tag.

## Anatomy of a 4.x service

```ts
// app/services/cart.ts
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

export default class CartService extends Service {
  @tracked items: CartItem[] = [];

  @cached
  get subtotal(): number {
    return this.items.reduce((s, i) => s + i.price * i.quantity, 0);
  }

  add(item: CartItem) { this.items = [...this.items, item]; }
  remove(id: string) { this.items = this.items.filter(i => i.id !== id); }
}

declare module '@ember/service' {
  interface Registry {
    cart: CartService;
  }
}
```

## Anatomy of a 4.x ember-data model

```ts
// app/models/post.ts
import Model, { attr, belongsTo, hasMany } from '@ember-data/model';
import type { AsyncBelongsTo, AsyncHasMany } from '@ember-data/model';
import type UserModel from './user';
import type CommentModel from './comment';

export default class PostModel extends Model {
  @attr('string') declare title: string;
  @attr('string') declare body: string;
  @attr('boolean', { defaultValue: false }) declare isDraft: boolean;

  @belongsTo('user', { async: true, inverse: 'posts' })
    declare author: AsyncBelongsTo<UserModel>;

  @hasMany('comment', { async: true, inverse: 'post' })
    declare comments: AsyncHasMany<CommentModel>;
}

declare module 'ember-data/types/registries/model' {
  export default interface ModelRegistry {
    post: PostModel;
  }
}
```

**4.x-specific:** the `inverse` and `async` options on relationships are now **errors if missing** (in late 4.x), not just deprecation warnings. If your 3.x → 4.x migration left them implicit, fix them all.

## Build pipelines available in 4.x

You'll see these in the wild, in roughly this order:

1. **Classic ember-cli** (default in early 4.x, still works through 4.12). Broccoli/Babel-based; no tree-shaking.
2. **Embroider compat** (`@embroider/core` + `@embroider/compat` + `@embroider/webpack`). Drop-in, tree-shakes static usage, prepares for optimized mode.
3. **Embroider optimized** (`staticHelpers: true`, `staticModifiers: true`, `staticComponents: true`, `staticAddonTrees: true`). Real ES module pipeline. Required for `<template>` tag in 5.x.

For a 4.x codebase you maintain through 2026, **adopt Embroider** at minimum in compat mode. It's a separate week of work, but it pays back in build performance and prepares the 5.x jump.

## Common 4.x mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Build fails with "Module not found: @ember/component" on a renamed file | A leftover `import Component from '@ember/component';` in code that should use `@glimmer/component`. | Replace the import. |
| `Cannot read properties of undefined (reading 'extend')` | An addon that was never modernized. | Bump the addon, or fork/inline. |
| `RangeError: Maximum call stack` on render | Classic two-way binding leftover (`(mut ...)` in a deeply nested template). | Replace with callbacks. |
| `Implicit injection deprecation` warnings | A `@service` is being used without an explicit name and the resolver guess is ambiguous. | `@service('foo') declare foo: FooService;` |
| `Test ended without all promises settled` | A custom waiter from 3.x not migrated to `@ember/test-waiters`. | Migrate the waiter or use `waitForPromise`. |
| Embroider build fails at "static check" | Dynamic component invocation by string at runtime. | Use `(component MyComponent)` (curried reference) instead of `(component "my-component")`. |

## Verification

- [ ] No `@ember/component` imports.
- [ ] No `Ember.X` global imports.
- [ ] No `(mut ...)` in templates.
- [ ] No `actions: {}` in classes.
- [ ] All ember-data relationships specify `async` and `inverse` explicitly.
- [ ] Tests on `module + setupXxxTest`.
- [ ] (Recommended) Embroider in compat mode at minimum.

## See also

- `ember-4-typescript-early` — adopting TS on 4.x.
- `ember-4-recommendations` — Embroider, addons, and the 5.x roadmap.
- `ember-4-to-5-migration` — the next jump.
- Modern reference: [`ember-octane-fundamentals`](../ember-octane-fundamentals).
