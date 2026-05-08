---
name: ember-octane-fundamentals
description: Core mental model for modern Ember (Octane edition, default since 3.15) — native classes, decorators, tracked properties, actions, owner/DI, autotracking. Use when writing or reviewing any Octane-era Ember code, when explaining how reactivity works, or when migrating from classic Ember.
type: reference
---

# Ember Octane Fundamentals

## What "Octane" means

Octane is the edition that became default in Ember 3.15 (December 2019). It is the baseline mental model for any modern Ember code. Polaris (the next edition) is additive — it changes *how you author* templates (template tag / strict mode) but the reactivity, DI, and lifecycle model below is unchanged.

If a repo uses `ember-source >= 3.15` and `@glimmer/component`, assume Octane.

## The five things that matter

1. **Native classes + decorators** replace `Ember.Object.extend({...})`.
2. **`@tracked` properties** are the reactivity primitive — change a tracked field, anything that read it re-renders.
3. **`@action`** binds a method's `this` so it can be invoked from a template without `.bind`.
4. **`@service`** is dependency injection — the container hands you a singleton.
5. **The owner** (`getOwner(this)`) is the container that resolves and instantiates everything.

You can write almost any Ember Octane app with just these five concepts.

## Native classes

```ts
// app/components/counter.ts
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';

interface CounterArgs {
  start?: number;
}

export default class Counter extends Component<{ Args: CounterArgs }> {
  @tracked count = this.args.start ?? 0;

  @action
  increment() {
    this.count += 1;
  }
}
```

```hbs
{{! app/components/counter.hbs }}
<button type="button" {{on "click" this.increment}}>
  {{this.count}}
</button>
```

Notes:
- `@glimmer/component` is the default component base. It has *no* `this.set`, no two-way bindings, and no classic lifecycle hooks (`didInsertElement`, etc.). Lifecycle is via modifiers (see `ember-components-and-templates`).
- `args` are read-only and reactive. Reassigning `this.args.foo = ...` throws.
- The class is a real ES class — `extends`, `super`, `static`, `#private` all work.

## `@tracked` — the reactivity primitive

A property marked `@tracked` is auto-observed by Glimmer. When you write to it, any template, getter, or helper that read it re-runs.

```ts
import { tracked } from '@glimmer/tracking';

class Cart {
  @tracked items: Item[] = [];

  get total(): number {
    return this.items.reduce((sum, i) => sum + i.price, 0);
  }

  add(item: Item) {
    // Replace the array — pushing in place is NOT tracked.
    this.items = [...this.items, item];
  }
}
```

### Tracking rules — memorize these

| Rule | Why |
|---|---|
| Mutating arrays/objects in place is **not** tracked | Tracking is per-property assignment. `arr.push(x)` doesn't reassign `arr`. |
| Replace the reference: `this.items = [...this.items, x]` | Or use `TrackedArray`/`TrackedMap` from [`tracked-built-ins`](https://github.com/tracked-tools/tracked-built-ins). |
| Getters that read tracked state are reactive automatically | No `@computed`, no dependency keys. |
| `@cached` memoizes a getter until its tracked deps change | Use it when the getter is expensive — see "Derived state" in `ember-services-and-state`. |
| Don't mutate `args` | They're upstream-owned. Pass callbacks back up via actions. |

### Common tracking mistake

```ts
// WRONG — mutation is invisible to the renderer
this.user.preferences.theme = 'dark';

// RIGHT — replace the property whose owner is tracked
this.user = { ...this.user, preferences: { ...this.user.preferences, theme: 'dark' } };

// OR — make the inner object a class with @tracked fields
class Preferences { @tracked theme = 'light'; }
```

## `@action`

`@action` is just a `this`-binding decorator. Use it on any method invoked from a template or passed as a callback:

```ts
import { action } from '@ember/object';

class Form extends Component {
  @action
  handleSubmit(event: SubmitEvent) {
    event.preventDefault();
    // `this` is the component, not the event target
  }
}
```

In templates:

```hbs
<form {{on "submit" this.handleSubmit}}>...</form>
```

Don't use `(action this.foo)` (the helper) in Octane — use `this.foo` directly with `{{on}}` and `fn`.

## Dependency injection — `@service`

Services are singletons resolved by the container. Inject them with `@service`:

```ts
import Component from '@glimmer/component';
import { service } from '@ember/service';
import type RouterService from '@ember/routing/router-service';
import type SessionService from 'ember-simple-auth/services/session';

export default class HeaderNav extends Component {
  @service declare router: RouterService;
  @service declare session: SessionService;

  get isOnDashboard() {
    return this.router.currentRouteName?.startsWith('dashboard');
  }
}
```

- `declare` (TS) tells the compiler the property exists without emitting an initializer that would shadow the decorator.
- The string name is inferred from the property (`router` → `service:router`). Override with `@service('shopping-cart') cart;` if needed.
- Services are also instantiable from routes, controllers, helpers, modifiers, and other services.

Define a service:

```ts
// app/services/shopping-cart.ts
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';

export default class ShoppingCartService extends Service {
  @tracked items: CartItem[] = [];

  add(item: CartItem) {
    this.items = [...this.items, item];
  }
}

declare module '@ember/service' {
  interface Registry {
    'shopping-cart': ShoppingCartService;
  }
}
```

The `Registry` augmentation gives `@service declare cart: ShoppingCartService;` proper typing.

## The owner and `getOwner`

The owner is the container. Every Ember object (component, route, controller, service) has one. You usually don't need it directly, but two cases come up:

1. **Manually instantiating a class that needs DI** (e.g. a class-based "resource" or POJO that should be able to inject services):

```ts
import { setOwner, getOwner } from '@ember/owner';

class PaymentProcessor {
  @service declare api: ApiService;
}

// inside a component / route:
const processor = new PaymentProcessor();
setOwner(processor, getOwner(this));
```

2. **Looking up a registered factory** at runtime (rare, mostly engines/addons):

```ts
const owner = getOwner(this);
const adapter = owner.lookup('adapter:application');
```

In Polaris-era code you'll see `import { getOwner } from '@ember/owner';` (the `'@ember/application'` import still works but is deprecated for this purpose).

## Lifecycle: there is no lifecycle

Glimmer components do not have `didInsertElement`, `willDestroyElement`, etc. Instead:

- DOM-side effects → **modifiers** (`{{did-insert}}`, `{{will-destroy}}`, or `ember-modifier` for custom).
- Async work → **`ember-concurrency`** tasks or `@ember/concurrency`-style flows.
- Cleanup tied to the component's life → `registerDestructor(this, fn)` from `@ember/destroyable`.

```ts
import { registerDestructor } from '@ember/destroyable';

export default class Listener extends Component {
  constructor(owner: any, args: object) {
    super(owner, args);
    const handler = () => { /* ... */ };
    window.addEventListener('resize', handler);
    registerDestructor(this, () => window.removeEventListener('resize', handler));
  }
}
```

## Convention over configuration

Ember's payoff comes from convention. Don't fight it.

| Convention | Where it shows up |
|---|---|
| File path = module name | `app/components/user-card.ts` → `<UserCard>` |
| Pods are deprecated | Use the classic `app/{components,routes,controllers,services,...}` layout. |
| Route file = URL segment | `app/routes/posts/show.ts` matches `/posts/:post_id` (when wired in `router.ts`). |
| Templates colocate with components | `app/components/user-card.{ts,hbs}` (or `.gjs` in Polaris). |
| Resolver finds factories by type:name | `service:session`, `route:posts.show`, `component:user-card`. |

If you find yourself building a registry, a side-loader, or a custom resolver — stop and check whether the Ember resolver already does it.

## Octane checklist for new code

- [ ] `@glimmer/component` (not `@ember/component`).
- [ ] `@tracked` for any state the template reads — not POJO fields.
- [ ] `@action` on every method passed to `{{on}}` or as a callback.
- [ ] `@service declare` (with `Registry` augmentation in TS) for every dependency.
- [ ] No `this.set` / `this.get` — use plain property access.
- [ ] No two-way `{{mut}}` — pass callbacks instead (data down, actions up).
- [ ] No classic component lifecycle hooks — use modifiers and `registerDestructor`.

## Red flags

- A component reassigning `this.args.x` (read-only).
- `@tracked` on getters (decorator goes on fields, not getters — use `@cached` on getters).
- `Ember.Object.extend({...})` in new code.
- `set(this, 'foo', x)` instead of `this.foo = x`.
- `didInsertElement` / `willDestroyElement` in `@glimmer/component` (not called).
- `this.toString()` parsing for type info — use `getOwner` and registry types.

## See also

- `ember-components-and-templates` — Glimmer components, args, modifiers, helpers.
- `ember-services-and-state` — `@cached`, derived state, app-level singletons.
- `ember-typescript-and-glint` — `Registry`, `@service declare`, signature types.
- `ember-polaris-migration` — `<template>` tag and strict mode.
