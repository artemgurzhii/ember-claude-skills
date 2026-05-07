---
name: ember-components-and-templates
description: Authoring Glimmer components and Handlebars templates in modern Ember — args, blocks, yield, helpers, modifiers, splattributes, contextual components. Use when creating, refactoring, or reviewing any Ember component or template.
type: reference
---

# Ember Components & Templates

Use this skill alongside `ember-octane-fundamentals` (which covers `@tracked`, `@action`, DI). This skill is about how components are *shaped* — args in, DOM out — and how templates compose.

## Component anatomy (colocated)

```
app/components/
  user-card.ts     # backing class (optional — pure templates are fine)
  user-card.hbs    # template
```

You can also write template-only components — just `user-card.hbs` with no `.ts`. They get `args` automatically.

```hbs
{{! app/components/user-card.hbs }}
<div class="user-card">
  <h3>{{@user.name}}</h3>
  <p>{{@user.email}}</p>
</div>
```

`@user` is shorthand for `this.args.user` inside a template.

## Args — the contract

Args are **read-only**, **reactive**, and **upstream-owned**. Never mutate them.

```ts
import Component from '@glimmer/component';

interface UserCardArgs {
  user: User;
  onEdit?: (user: User) => void;
}

export default class UserCard extends Component<{
  Args: UserCardArgs;
  Element: HTMLDivElement;     // for splattributes — see below
  Blocks: {
    default: [];
    actions: [{ edit: () => void; delete: () => void }];
  };
}> {
  @action handleEdit() {
    this.args.onEdit?.(this.args.user);
  }
}
```

The `Signature` (the type parameter) is what Glint uses to type-check templates. See `ember-typescript-and-glint`.

## Calling components

Angle-bracket invocation is the modern form:

```hbs
<UserCard @user={{this.currentUser}} @onEdit={{this.handleEdit}} />

{{!-- Curly invocation still works but is legacy: --}}
{{user-card user=this.currentUser onEdit=this.handleEdit}}
```

Use angle brackets in all new code. They:
- Make the args explicit (`@foo`).
- Allow HTML attributes that pass through to the root element via `...attributes`.
- Match Polaris template tag syntax.

## `...attributes` (splattributes)

`...attributes` forwards every HTML attribute and modifier from the invocation site to a chosen element:

```hbs
{{! app/components/button.hbs }}
<button type="button" class="btn" ...attributes>
  {{yield}}
</button>
```

```hbs
<Button class="btn-primary" data-test-save {{on "click" this.save}}>
  Save
</Button>
```

The `class` merges, `data-test-save` is added, and the `{{on "click"}}` modifier is attached — all to the root `<button>`.

Rules:
- **Use `...attributes` exactly once** (typically on the root element).
- Put it **last** so consumer attributes win over defaults.
- The `Element` field in the component signature is the type of the element that receives `...attributes`.

## Blocks and `{{yield}}`

A block is a chunk of template the parent passes in:

```hbs
{{! app/components/modal.hbs }}
<dialog ...attributes>
  <header>{{yield to="header"}}</header>
  <section>{{yield}}</section>
  <footer>{{yield this.close to="footer"}}</footer>
</dialog>
```

```hbs
<Modal as |close|>
  <:header>Confirm</:header>
  Are you sure?
  <:footer as |close|>
    <button {{on "click" close}}>Cancel</button>
  </:footer>
</Modal>
```

- `{{yield}}` (no `to`) is the **default** block.
- `{{yield to="name"}}` is a **named** block.
- `{{yield value1 value2}}` exposes block params (`as |a b|`).
- `{{has-block "name"}}` and `{{has-block-params}}` let you render fallbacks.

## Helpers

Helpers are pure functions invoked in templates.

```ts
// app/helpers/format-currency.ts
import { helper } from '@ember/component/helper';

export default helper(function formatCurrency(
  [amount]: [number],
  { currency = 'USD' }: { currency?: string }
) {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount);
});
```

```hbs
{{format-currency @order.total currency="EUR"}}
```

For helpers that need DI (e.g. read a service), use a class helper:

```ts
import Helper from '@ember/component/helper';
import { service } from '@ember/service';
import type IntlService from 'ember-intl/services/intl';

export default class THelper extends Helper {
  @service declare intl: IntlService;

  compute([key]: [string], hash: Record<string, unknown>) {
    return this.intl.t(key, hash);
  }
}
```

In Polaris (template tag) most helpers are just imported functions — no `helper()` wrapper needed (see `ember-polaris-migration`).

## Modifiers

Modifiers run code on a DOM element. Built-ins:

| Modifier | Purpose |
|---|---|
| `{{on "event" handler}}` | DOM event listener (replaces `{{action}}`). |
| `{{did-insert this.fn}}` | Run when the element is inserted (`@ember/render-modifiers`). |
| `{{did-update this.fn dep1 dep2}}` | Run when args change. |
| `{{will-destroy this.fn}}` | Run before the element is removed. |

Custom modifiers via [`ember-modifier`](https://github.com/ember-modifier/ember-modifier):

```ts
// app/modifiers/auto-focus.ts
import { modifier } from 'ember-modifier';

export default modifier((element: HTMLElement) => {
  element.focus();
});
```

```hbs
<input {{auto-focus}} />
```

For modifiers with cleanup or args:

```ts
import { modifier } from 'ember-modifier';

export default modifier((el: HTMLElement, _: [], { onResize }: { onResize: () => void }) => {
  const observer = new ResizeObserver(onResize);
  observer.observe(el);
  return () => observer.disconnect();
});
```

`@ember/render-modifiers` (`{{did-insert}}`, etc.) are an **escape hatch**, not the primary tool. Reach for a real modifier when the behavior belongs to the element, not the component.

## Conditionals and iteration

```hbs
{{#if @user.isAdmin}}
  <AdminPanel />
{{else if @user.isMember}}
  <MemberDashboard />
{{else}}
  <Login />
{{/if}}

{{#each @items as |item index|}}
  <Item @item={{item}} @index={{index}} />
{{else}}
  <p>No items yet.</p>
{{/each}}

{{#each-in @counts as |key value|}}
  {{key}}: {{value}}
{{/each-in}}

{{#let (capitalize @name) as |formattedName|}}
  Hi, {{formattedName}}.
{{/let}}
```

Use `{{#each}}` with `key="id"` (or another stable key) when items have identity to keep DOM diffs stable.

## Composing actions: `{{fn}}` and `(hash ...)` / `(array ...)`

```hbs
<button {{on "click" (fn this.delete @item.id)}}>Delete</button>

<UserList @users={{this.users}} @options={{hash sortBy="name" page=1}} />
```

- `{{fn cb arg1}}` is partial application — the equivalent of `cb.bind(null, arg1)`.
- `(hash ...)` and `(array ...)` build literals inside templates.

## Truth helpers

`ember-truth-helpers` (or `@ember/truth-helpers` in newer versions) ships:

```hbs
{{#if (and @isLoggedIn (or @isAdmin (eq @user.role "owner")))}} ... {{/if}}
{{#if (not @items.length)}}<EmptyState />{{/if}}
```

Without these you can't combine boolean expressions in templates. Most apps install them on day one.

## Contextual components — exposing a "namespace" of components

```hbs
{{! app/components/tabs.hbs }}
<div class="tabs" ...attributes>
  {{yield (hash
    Tab=(component "tabs/tab" activeId=this.activeId onSelect=this.select)
    Panel=(component "tabs/panel" activeId=this.activeId)
  )}}
</div>
```

```hbs
<Tabs as |t|>
  <t.Tab @id="overview">Overview</t.Tab>
  <t.Tab @id="settings">Settings</t.Tab>

  <t.Panel @id="overview">...</t.Panel>
  <t.Panel @id="settings">...</t.Panel>
</Tabs>
```

This is the standard pattern for "wrapper exposing a sub-API" (forms, dropdowns, lists). In Polaris this becomes plain JS imports of components.

## When to use a component vs a helper vs a modifier

| Use a... | When |
|---|---|
| Component | The thing renders DOM and may have state. |
| Helper | The thing transforms data and returns a value (string, array, hash). No DOM. |
| Modifier | The thing reaches into a DOM element to attach listeners, focus, observe. No DOM rendering. |

If you're writing a "helper" that does `document.querySelector(...)` — it should be a modifier. If you're writing a "component" with no template and no state — it should be a helper.

## Performance notes

- `<input @value={{this.foo}}>` does *not* two-way bind. Use `<input value={{this.foo}} {{on "input" this.update}}>` or [`ember-on-helper`](https://github.com/buschtoens/ember-on-helper) patterns.
- Avoid creating new objects/arrays in every render — `(hash a=1)` re-evaluates each render. For large lists, build the data in JS once.
- `{{#each}}` with `key` avoids rerendering identical rows.
- `(component "name" ...)` is fine; it produces a curried component reference, not a new component class each render.

## Common mistakes

| Mistake | Fix |
|---|---|
| Mutating `@arg.something` | Pass a callback up via `@onChange` instead. |
| `class="foo {{this.cls}}"` then `{{!-- ... --}}` then `class="bar"` | Class **must** appear once on each element. Use a single attribute and concatenate. |
| Using `{{action ...}}` | Switch to `{{on "click" this.fn}}` and `{{fn this.fn arg}}`. |
| Forgetting `this.` | `{{foo}}` is a helper or arg lookup. `{{this.foo}}` is the backing class property. |
| Calling `{{component "x"}}` to "render dynamic component" | Use `<this.dynamicCmp />` or `(component ...)` curried into a value. |

## Verification

- [ ] `...attributes` appears exactly once, on the root element, last.
- [ ] No two-way bindings; data flows down via args, actions flow up via callbacks.
- [ ] `@action` on every method passed via `{{on}}` or `{{fn}}`.
- [ ] No `Ember.Component` / `@ember/component` in new code (use `@glimmer/component`).
- [ ] Component signatures (`Args`, `Element`, `Blocks`) declared in TS.

## See also

- `ember-octane-fundamentals` — `@tracked`, `@action`, lifecycle.
- `ember-typescript-and-glint` — component signature types.
- `ember-polaris-migration` — `<template>` tag and importing components/helpers/modifiers directly.
- `ember-ecosystem-addons` — `ember-modifier`, `ember-truth-helpers`, `ember-render-modifiers`.
