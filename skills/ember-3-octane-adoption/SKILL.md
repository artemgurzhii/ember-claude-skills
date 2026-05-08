---
name: ember-3-octane-adoption
description: Adopting Octane within an Ember 3.16+ app — flipping the optional features, running ember-octane-codemods (native classes, tracked, angle brackets, no-implicit-this), per-file conversion order, and the addons that unlock the new mental model. Use when transitioning a 3.x app from classic to Octane without changing the Ember version, when scoping the conversion effort, or when reviewing an Octane-conversion PR.
type: reference
---

# Ember 3.x — Adopting Octane

You can fully adopt Octane *while staying on Ember 3.x*. The Octane mental model — `@glimmer/component`, `@tracked`, modifiers, native classes — is the same one used in 4.x, 5.x, and Polaris. Doing the conversion now means the eventual major-version bumps become near-trivial.

This skill is the structured plan for converting a classic-3.x app to Octane-3.x. After this, the [`ember-3-to-4-migration`](../ember-3-to-4-migration) skill takes over.

## Prerequisites

- Ember 3.16+ (3.20+ is much smoother).
- jQuery integration disabled (`@ember/optional-features`):
  ```bash
  pnpm exec ember feature:disable jquery-integration
  ```
- `@glimmer/component` and `@glimmer/tracking` installed:
  ```bash
  pnpm add @glimmer/component @glimmer/tracking
  ```
- `@ember/render-modifiers` and `ember-modifier` installed (for lifecycle bridging).
- Tests on the modern API (`module + setupRenderingTest`). Some Octane patterns can't be expressed in the classic test API.

## Step 1 — Flip the optional features

`config/optional-features.json`:

```json
{
  "application-template-wrapper": false,
  "default-async-observers": true,
  "jquery-integration": false,
  "template-only-glimmer-components": true
}
```

Each flag, in plain English:

| Flag | What it does |
|---|---|
| `application-template-wrapper: false` | The application's `<div class="ember-application">` wrapper is gone. The template renders directly into the element you mounted into. |
| `default-async-observers: true` | Observers fire async (closer to render) instead of sync. Required for tracked-friendly behavior. |
| `jquery-integration: false` | jQuery isn't bundled. `this.$()` and `Ember.$` go away. |
| `template-only-glimmer-components: true` | A template-only component (`foo.hbs` with no `foo.js`) renders as a Glimmer component (no wrapper element), not a classic one. **This is the riskiest flag** — every layout assumption based on the wrapper `<div>` may need fixing. |

Flip them **one at a time** in 3.x. Run the suite after each flip. The `template-only-glimmer-components` flip is the one that surfaces hidden CSS/layout dependencies — schedule a designer pass for that day.

## Step 2 — Run the Octane codemods

The umbrella tool is [`ember-octane-codemods`](https://github.com/ember-codemods/ember-octane-codemods). Under the hood it's a sequence:

```bash
pnpm exec ember-cli-update --run-codemods
```

The order matters — and is typically:

1. **`ember-no-implicit-this-codemod`** — `{{foo}}` → `{{this.foo}}` for class-field references.
2. **`ember-angle-brackets-codemod`** — `{{user-card}}` → `<UserCard>`.
3. **`ember-on-modifier-codemod`** — `onclick={{action "save"}}` → `{{on "click" this.save}}`.
4. **`ember-native-class-codemod`** — `Component.extend({...})` → `class extends Component { ... }` with decorators.
5. **`ember-tracked-properties-codemod`** — `computed`, `set/get` calls, observers → `@tracked` + getters.

Run each, commit, run tests. Don't run all five at once.

## Step 3 — Per-file conversion order (manual)

Codemods get you 70% of the way. The remaining 30% is per-file judgment. Order:

```
1. Pure-template components (no .js file)        ← already Glimmer after step 1
2. Leaf components with simple state              ← @tracked + @action
3. Components with @ember/render-modifier hooks   ← keep as-is or extract real modifiers
4. Components with mixins                         ← extract mixin to service first
5. Components with computed properties + @each    ← convert to getters reading @tracked
6. Components with classic actions: {...}         ← @action methods
7. Layouts / app-wrappers                         ← last; they touch everything
```

For each file:

1. Convert the class to `class extends Component`.
2. Replace `extend({})` second-arg behavior with class fields/methods.
3. Add `@tracked` to anything the template reads or that gets set.
4. Add `@action` to methods invoked from templates or as callbacks.
5. Replace `actions: { foo() {} }` with `@action foo() {}`.
6. Replace `this.set('foo', x)` with `this.foo = x`. (If that breaks reactivity, the field needs `@tracked`.)
7. Remove `_super(...arguments)` — it's classic-only.
8. If converting from `@ember/component` → `@glimmer/component`, remove `tagName`/`classNames`/`classNameBindings`/`attributeBindings` and use `...attributes` on the chosen root element of the template.

Do **not** mix dialects in one file. If you can't finish a conversion in the PR, leave the file classic.

## Step 4 — Services first, components later (counter-intuitive but right)

Convert services to Octane *before* the components that use them. Why: an Octane component reading a non-tracked field on a classic service won't update. Once services are Octane, *both* classic and Octane consumers see updates.

```ts
// app/services/cart.ts
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';

export default class CartService extends Service {
  @tracked items = [];
  add(item) { this.items = [...this.items, item]; }
}
```

The classic consumers (`Component.extend({ cart: service(), ... })`) keep working — the service's `@tracked` field is observable through both APIs.

## Step 5 — `@cached` for derived state

Once `@tracked` lands, getters that read tracked state are reactive. For expensive getters, add `@cached` from `@glimmer/tracking`:

```ts
import { cached, tracked } from '@glimmer/tracking';

class TodoList {
  @tracked todos = [];

  @cached
  get sortedByDeadline() {
    return [...this.todos].sort((a, b) => a.deadline - b.deadline);
  }
}
```

## Step 6 — The mixin extraction pass

Run this as a focused pass after the leaf components are converted:

1. List every mixin in `app/mixins/`.
2. For each: identify the consumers, group by intent.
3. Extract to a service when the behavior is "shared state or shared dependency" (logging, audit, polling).
4. Extract to a utility module when the behavior is "shared pure logic" (formatters, calculators).
5. Extract to a class decorator only as a last resort.
6. Convert each consumer site, one PR at a time.

Mixins are the single biggest blocker to a clean Octane state. Plan a quarter-week for this if you have many.

## Step 7 — Modifier hygiene

After the codemods, you'll have a lot of `{{did-insert this.fn}}` calls courtesy of `@ember/render-modifiers`. That's fine as a transient state; long-term, write **custom modifiers** for the recurring patterns:

```ts
// app/modifiers/auto-focus.ts
import { modifier } from 'ember-modifier';
export default modifier((el) => el.focus());
```

```hbs
{{!-- before --}}
<input {{did-insert this.focus}} />

{{!-- after --}}
<input {{auto-focus}} />
```

Aim to make `{{did-insert}}` rare, used only for one-off page-init hooks.

## Step 8 — Routes and controllers can wait

Octane gives most of its benefits in components and services. Routes barely change. Controllers change only in that you'll narrow them to query-param holders.

A reasonable line: **leave routes and controllers in classic-syntax `extend({...})` form** through 3.x, convert them to `class extends Route { ... }` during the 4.x migration. Doing it earlier pays off less and risks regressions in transition logic.

## Step 9 — Templates: angle brackets and `(fn)` and `{{on}}`

After the codemods, lock in:

- All component invocations in angle brackets.
- All event handlers as `{{on "event" this.handler}}` — never `onclick={{action ...}}`.
- Callbacks with arguments use `(fn this.handler arg)` — never `(action ...)`.
- All references to class fields in templates start with `this.`.

Add a `template-lint` rule pass to catch regressions:

```bash
pnpm exec ember-template-lint --config-path .template-lintrc.js .
```

`.template-lintrc.js`:

```js
module.exports = {
  extends: ['octane'],
};
```

## Step 10 — Component test conversion

Every component you Octane-port should also have its test file ported to `setupRenderingTest`. In 3.x, the matching test API is fully available, so do them in the same PR.

```ts
// before
moduleForComponent('user-card', 'Integration | Component | user-card', { integration: true });

// after
module('Integration | Component | user-card', function (hooks) {
  setupRenderingTest(hooks);
  test('...', async function (assert) {
    await render(hbs`<UserCard @user={{this.user}} />`);
    assert.dom('[data-test-name]').hasText('Ada');
  });
});
```

## What you do *not* do during Octane adoption

- **Don't introduce TypeScript** in the same window. TS+Octane is great; doing both in one go triples the risk.
- **Don't introduce Embroider.** Embroider arrives properly in 4.x. Trying to bolt it on during Octane conversion is a separate, riskier project.
- **Don't remove ember-data.** Octane works fine with classic ember-data. Save the ember-data refactor for the 4.x window.
- **Don't ship template tag (`<template>`)** — that's Polaris-era. 3.x is not the time.
- **Don't refactor architecturally.** Convert syntax. Save architectural changes for after the upgrade is done.

## Verification — am I "Octane-adopted"?

- [ ] All optional features set as listed above.
- [ ] `template-lint` passes the `octane` ruleset.
- [ ] No `Ember.Component.extend(...)` or `@ember/component` imports in components.
- [ ] No `actions: {...}` hashes; all callbacks are `@action`.
- [ ] All event handling uses `{{on}}` and `(fn ...)`.
- [ ] All component invocations are angle brackets.
- [ ] All class-field template references use `this.`.
- [ ] Mixins extracted to services or utilities.
- [ ] `@ember/render-modifiers` uses are documented; recurring patterns are real custom modifiers.
- [ ] Tests for converted components on `setupRenderingTest`.
- [ ] No `(mut foo)` in templates.

When that's all green, you're ready to start the [3.28 → 4.x jump](../ember-3-to-4-migration).

## See also

- `ember-3-mixed-classic-octane` — the patchwork state during conversion.
- `ember-3-to-4-migration` — the next step.
- Modern reference: [`ember-octane-fundamentals`](../ember-octane-fundamentals).
