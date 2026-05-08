---
name: ember-3-mixed-classic-octane
description: Reading and maintaining Ember 3.x apps where classic and Octane patterns coexist — half the components are Ember.Component.extend, the other half are @glimmer/component, and tests run on both APIs. Use when triaging "why does this file look different from that file," when reviewing a partial-migration PR, or when planning the order of further conversion.
type: reference
---

# Ember 3.x — The Mixed Classic+Octane Reality

Most real Ember 3.x codebases are a patchwork. The pattern is so common it deserves its own skill: in any 3.16+ app you'll see, on the same day:

- A `app/components/legacy-table.js` written as `Ember.Component.extend({...})`.
- A `app/components/user-card.ts` written as `class UserCard extends Component { ... }` with `@tracked`.
- A test file using `moduleForComponent('legacy-table', { integration: true })`.
- Another test file using `module(..., function (hooks) { setupRenderingTest(hooks); })`.

This is fine. Both shapes are supported in 3.x simultaneously. But you need to read both fluently and know how they interact.

## Decision tree when adding code to a 3.x app

```
Is the surrounding file Octane (class extends Component)?
  → Add Octane code. Don't mix dialects in one file.

Is the surrounding file classic (Ember.Component.extend)?
  → Match the file. Don't half-port a single component during a feature change.

Are you creating a new file?
  → Use Octane. New files should look like the destination state.
```

The rule is: **convert at the file boundary, never mid-file**.

## Components — what each shape looks like in the same app

```js
// app/components/legacy-table.js (classic)
import Component from '@ember/component';
import { computed } from '@ember/object';

export default Component.extend({
  classNames: ['table'],
  rows: null,
  sortKey: 'name',

  sorted: computed('rows.@each.{name,date}', 'sortKey', function () {
    return [...(this.rows || [])].sort((a, b) =>
      a[this.sortKey] > b[this.sortKey] ? 1 : -1
    );
  }),

  actions: {
    setSort(key) { this.set('sortKey', key); }
  }
});
```

```ts
// app/components/user-card.ts (Octane)
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';

interface Signature {
  Element: HTMLDivElement;

  Args: {
    user: User;
  }
}

export default class UserCard extends Component<Signature> {
  @tracked isExpanded = false;

  @action toggle() { this.isExpanded = !this.isExpanded; }
}
```

Both ship in the same build. The classic component still wraps with a `<div>` (because `@ember/component` does); the Octane component has no wrapper.

## Templates — invocation works both ways

You can mix angle-bracket and curly invocation in the same template, and you can call a classic component with angle brackets:

```hbs
{{!-- both fine in 3.x --}}
<LegacyTable @rows={{this.rows}} />
{{legacy-table rows=this.rows}}

{{!-- both fine in 3.x --}}
<UserCard @user={{this.currentUser}} />
{{user-card user=this.currentUser}}
```

Default to **angle brackets in new templates**, even when calling classic components. The codemod (`ember-angle-brackets-codemod`) is the official path for the rest of the app.

## Two-way binding — the half-migrated trap

A classic component may rely on `(mut foo)` from its parent. If you upgrade the *parent* to Octane but leave the child classic, the binding silently breaks:

```hbs
{{!-- parent (Octane) --}}
<LegacyForm @value={{(mut this.draft)}} />
```

`(mut ...)` is deprecated in Octane and produces unreliable behavior across the boundary. Two safe paths:

1. **Pass a callback instead** — port both ends to one-way data flow:
   ```hbs
   <LegacyForm @value={{this.draft}} @onChange={{this.setDraft}} />
   ```
2. **Keep `mut` on both sides until you can convert the child too** — only acceptable as a temporary state.

When in doubt, port the child component to Octane in the same PR.

## Computed → tracked — interop rules

`@tracked` and `computed(...)` are mutually visible in 3.x. A classic computed property can read a `@tracked` field on another object, and an Octane getter can read a classic computed property. The rendering pipeline handles both.

But: **classic `set(this, 'foo', x)` does NOT trigger Octane re-renders** if `foo` is a regular field. Always use `@tracked` on Octane class fields, even when consumers might be classic components.

```ts
class CartService extends Service {
  @tracked items = [];          // ✅ classic and Octane consumers both see updates
  count = 0;                    // ❌ classic set(this, 'count', x) won't notify Octane templates
}
```

## Mixins — the cross-paradigm tax

Classic mixins **cannot be applied** to `@glimmer/component` or to ES classes that don't use `extend(...)`. Common interop options:

1. **Service-ify the mixin.** Move shared behavior into a service injected at the consumer.
2. **Composable function.** Plain JS functions called from the consumer's constructor or actions.
3. **Class decorator.** A custom decorator that adds methods/fields. Avoid unless you really need it — decorators are powerful but invisible.

Don't try to teach a mixin to work with classes. The investment is wasted; the destination has no mixins.

## Lifecycle — modifier substitutes

Classic components have `didInsertElement` / `willDestroyElement`. Octane `@glimmer/component` has neither. During the mixed era, the canonical bridges:

- **`@ember/render-modifiers`**: provides `{{did-insert this.fn}}`, `{{will-destroy this.fn}}`, `{{did-update this.fn}}`. Drop-in replacement for the most common lifecycle uses.
- **`ember-modifier`**: write a real modifier when the behavior belongs to the element (focus, observers, plugin integration).

```hbs
{{!-- bridge in a converting component --}}
<input
  {{did-insert this.focusOnMount}}
  {{will-destroy this.cleanupHandlers}}
/>
```

Treat `@ember/render-modifiers` as a **migration tool**, not a long-term answer. After conversion, refactor the most common usages into custom modifiers.

## Services across the boundary

Services are the cleanest interop point. A classic component injects via:

```js
export default Component.extend({
  session: service(),
  init() {
    this._super(...arguments);
    if (this.session.isAuthenticated) { /* ... */ }
  }
});
```

An Octane component injects via:

```ts
@service declare session: SessionService;
```

Both reach the same singleton. Services should be **defined as Octane classes** (with `@tracked` state) regardless of where they're consumed — classic consumers will pick up the reactivity correctly.

## Tests — both styles, same suite

`ember-qunit@^4` and later support `moduleFor*` and the modern `module/setupTest` API in the same suite. New tests should be modern; old tests can wait.

```js
// Old-style — keep until convenient to convert
import { moduleForComponent, test } from 'ember-qunit';
moduleForComponent('legacy-table', 'Integration | Component | legacy-table', { integration: true });
test('renders rows', function (assert) {
  this.set('rows', [{ name: 'a' }]);
  this.render(hbs`{{legacy-table rows=rows}}`);
  assert.equal(this.$('.row').length, 1);
});
```

```ts
// New-style — use for everything new
import { module, test } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';
import { hbs } from 'ember-cli-htmlbars';

module('Integration | Component | user-card', function (hooks) {
  setupRenderingTest(hooks);
  test('renders the name', async function (assert) {
    this.set('user', { name: 'Ada' });
    await render(hbs`<UserCard @user={{this.user}} />`);
    assert.dom('[data-test-name]').hasText('Ada');
  });
});
```

## Reading mixed code — what to look for

| Marker | What it tells you |
|---|---|
| `import Component from '@ember/component'` | Classic. Has a wrapper element. Use `_super(...arguments)`. |
| `import Component from '@glimmer/component'` | Octane. No wrapper. No lifecycle hooks. |
| `extend({...})` vs `class X extends Component` | Same distinction. |
| `Ember.computed(...)` or `computed(...)` | Classic dep-keyed reactivity. |
| `@tracked` / `@cached` | Octane reactivity. |
| `actions: { foo() {} }` + `{{action "foo"}}` | Classic action handling. |
| `@action foo() {}` + `{{on "click" this.foo}}` | Octane action handling. |
| `this.$()` | jQuery (only in classic; will break once jQuery is removed). |
| `(mut foo)` | Two-way binding. Octane-incompatible. |
| `Ember.Object.extend({...})` for non-components | A POJO-ish class that participates in the property system. Convert to `class` + `@tracked`. |

## Common mixed-mode bugs

| Symptom | Cause | Fix |
|---|---|---|
| Octane child doesn't re-render when classic parent updates | Parent stored state on a non-tracked field. | Make the parent's state `@tracked` (or convert it to a service). |
| `set(this, 'foo', x)` works in tests but not in prod | Octane consumer reading a non-tracked field via a getter. | Add `@tracked` to the field. |
| Classic component crashes after Octane parent removes a `(mut x)` | Parent dropped the writable cell, child still expects it. | Pass `(mut x)` until the child is converted, or convert the child. |
| `didInsertElement` never fires after class conversion | The component became `@glimmer/component`. | Use `{{did-insert}}` from `@ember/render-modifiers` (interim) or write a real modifier. |
| `mixin` errors on `@glimmer/component` | Mixins don't compose with class-style components. | Service-ify the mixin. |
| Test passes locally, fails in CI after conversion | An `andThen` block was left in the converted file. | Replace with `await` on the preceding helper. |

## Conversion order — what to port first

1. **Leaf components first.** Components that don't compose other components have the smallest surface.
2. **Stateful but isolated components next.** Anything reading services and rendering — they convert cleanly.
3. **Container/layout components last.** They orchestrate children; convert when the children are stable.
4. **Routes and controllers stay classic-style longer.** Octane's biggest gains are in components and services. Routes don't gain much from conversion until 4.x.
5. **Mixins go away as you convert their consumers.** Don't pre-extract.

A good heuristic: when you'd touch a file for an unrelated bug, port the surrounding component to Octane in the same PR — so long as the diff is reviewable.

## Verification

- [ ] You can read both component shapes without translating in your head.
- [ ] No file mixes dialects internally (no `extend({...})` and `@tracked` in the same class).
- [ ] Services define state with `@tracked`, regardless of consumer dialect.
- [ ] `(mut foo)` only appears between two classic components that both still need it.
- [ ] `@ember/render-modifiers` is installed; uses are itemized and scheduled for refactor to custom modifiers.
- [ ] New tests use `module + setupRenderingTest`; old tests have a planned conversion order.

## See also

- `ember-3-octane-adoption` — the structured Octane migration within 3.x.
- `ember-3-recommendations` — settling on 3.28 LTS as a checkpoint.
- `ember-3-to-4-migration` — the next jump.
