---
description: Review the current branch (or staged changes) against Ember best-practices — flag classic-pattern regressions, (mut ...) usage, find()-in-assertions, missing test selectors, ember-data mutation anti-patterns, and Octane/Polaris hygiene.
---

# /ember-review

Audit the current branch's diff against the Ember conventions this plugin enforces. Flag issues by severity, propose the smallest patch for each.

## Usage

```
/ember-review [--base=<ref>] [--staged] [--severity=<min>]
```

- `--base=<ref>`: compare against this ref. Default is the merge-base with `main`.
- `--staged`: only review files in `git diff --cached`.
- `--severity=<min>`: filter output to `error` (default) | `warn` | `info`.

## Steps

1. **Detect Ember version.** Read `ember-source` from `package.json` to know which conventions apply (e.g. classic components are *deprecated* on 3.x, *removed* on 4.x — same finding, different severity).
2. **List changed files** with `git diff --name-only <base>...HEAD`. Restrict review to:
   - `app/**/*.{js,ts,gjs,gts,hbs}`
   - `addon/**/*.{js,ts,gjs,gts,hbs}` (for addons)
   - `tests/**/*.{js,ts,gjs,gts,hbs}`
   - `package.json` (deps changed)
3. **Run the lint matrix below** on each file. For each finding, capture: file path, line number, rule id, severity, one-line diagnosis, smallest patch.
4. **Group by severity, then by rule.** Output errors first.
5. **End with a verdict** — green (no errors), yellow (errors that are auto-fixable), red (errors needing design discussion).

## Lint matrix

### Component & template hygiene

| Rule | Pattern | Severity | Fix |
|---|---|---|---|
| `no-classic-component` | `import Component from '@ember/component'` in new/changed files | error (4.x+), warn (3.x) | Convert to `@glimmer/component` + `<template>` or `.hbs`. |
| `no-mut-helper` | `(mut ` in any `.hbs` / `.gjs` / `.gts` | error | Replace with an `@action` setter and pass `this.foo` as the callback. |
| `no-action-modifier` | `{{action ...}}` in templates | error (4.x+), warn (3.x) | Use `{{on "event" this.handler}}` modifier. |
| `no-classic-actions-hash` | `actions: { ... }` block in a class body | error | Convert each entry to an `@action` method. |
| `prefer-on-modifier` | `onclick=` / `onsubmit=` attributes in templates | warn | Use `{{on "click" this.handler}}`. |
| `no-attr-in-template` | `{{attrs.foo}}` (classic component access) | error | Replace with `{{@foo}}`. |
| `prefer-spread-attributes` | Component template root element without `...attributes` | warn | Add `...attributes` to the root element so callers can pass `class`, `data-*`, `aria-*`. |
| `template-only-no-class` | Backing class with no fields/methods exists alongside template | info | Delete the class; mark as template-only. |

### State & reactivity

| Rule | Pattern | Severity | Fix |
|---|---|---|---|
| `no-pojo-mutation` | `this.someObj.field = ...` where `someObj` is a plain POJO read by templates | error | Use a `@tracked` field, or `tracked-built-ins`, or assign a new object. |
| `no-array-push-in-tracked` | `.push(`/`.pop(`/`.splice(` on a `@tracked` array | error | Spread into a new array, or use `TrackedArray` from `tracked-built-ins`. |
| `prefer-cached-getter` | Expensive getter (loops/sorts > N) without `@cached` | warn | Add `@cached`. |
| `no-controller-state` | New `@tracked` fields added to a `Controller` for cross-route data | warn | Move to a service. |
| `no-getOwner-in-product` | `getOwner(this).lookup('service:...')` in app code | error | Use `@service declare foo: FooService`. |

### Routing & data

| Rule | Pattern | Severity | Fix |
|---|---|---|---|
| `no-renderTemplate` | `renderTemplate` hook in a route | error (4.x+), warn (3.x) | Use named outlets in the parent template or refactor the route hierarchy. |
| `no-willTransition` | `willTransition` action in a route | warn | Listen to `routeWillChange` on the `RouterService`. |
| `model-hook-no-side-effects` | `model()` mutates a service or fires analytics | warn | Move to `afterModel` or a route-action. |
| `prefer-await-in-model` | `model()` returns a promise chain rather than `async/await` | info | Convert to `async model()`. |

### Tests

| Rule | Pattern | Severity | Fix |
|---|---|---|---|
| `no-find-for-assertions` | `find('[data-test-...]')` from `@ember/test-helpers` followed by `assert.equal/notEqual` on properties | error | Use `assert.dom(...)` (qunit-dom) or an `ember-cli-page-object` accessor. |
| `no-css-class-selectors` | Test selector targets `.classname` rather than `[data-test-*]` | error | Add `data-test-*` to the template, switch the test to it. |
| `no-then-chain-in-tests` | `visit('/x').then(...)` instead of `await visit('/x')` | error | Use `await`. |
| `no-setTimeout-in-tests` | `setTimeout(`/`Promise.resolve().then(` inside a test body | error | Use `waitFor` from `@ember/test-helpers` or wrap source-side work in `@ember/test-waiters`. |
| `prefer-page-object` | Same `[data-test-*]` selector repeated across 3+ tests | warn | Extract to an `ember-cli-page-object`. |
| `no-classic-test-api` | `moduleFor`, `moduleForComponent`, `moduleForAcceptance`, `andThen` | error (4.x+), warn (3.x) | Use `setupTest`/`setupRenderingTest`/`setupApplicationTest`. |

### Type safety (TS / Glint projects)

| Rule | Pattern | Severity | Fix |
|---|---|---|---|
| `service-registry-missing` | `export default class FooService extends Service` without a `declare module '@ember/service' { interface Registry { ... } }` block | error | Add the Registry augmentation. |
| `component-signature-missing` | `class extends Component` without a `Signature` type parameter | warn | Add `interface FooSignature { Args; Element; Blocks }` and pass it. |
| `any-in-args` | `Args: any` or untyped `args` access | warn | Type `Args` explicitly. |

### Dependencies

| Rule | Pattern | Severity | Fix |
|---|---|---|---|
| `removed-jquery-import` | `import $ from 'jquery'` in app/addon code | error (4.x+) | Use vanilla DOM or a modifier. |
| `unmaintained-addon` | `package.json` adds an addon last published > 3y ago | warn | Check for a maintained fork or an in-app replacement. |
| `addon-not-octane-ready` | Addon known not to support Octane | warn | Check `ember-ecosystem-addons` for the modern equivalent. |

## Output format

```
ember-review — base: main (sha abc1234), 14 files changed
ember-source: 5.4.1

ERRORS (3)
──────────────────────────────────────────────────────────────
[no-mut-helper]  app/components/cart-row.hbs:12
  {{input value=(mut @item.quantity)}}
  → Replace with:
    {{on "input" (fn this.setQuantity @item)}}

[no-find-for-assertions]  tests/integration/components/cart-row-test.ts:24
  let el = find('[data-test-quantity-input]');
  assert.equal(el.value, '3');
  → assert.dom('[data-test-quantity-input]').hasValue('3');

[service-registry-missing]  app/services/checkout.ts:1
  → Add at the bottom of the file:
    declare module '@ember/service' {
      interface Registry { checkout: CheckoutService; }
    }

WARNINGS (2)
──────────────────────────────────────────────────────────────
[no-controller-state]  app/controllers/cart.ts:7
  Controllers persist across navigations. Move @tracked discountCode to a service.

[unmaintained-addon]  package.json
  ember-cli-flash@4.0.0 last published 2021-08. Consider ember-toast.

VERDICT: 🔴 3 errors — please address before merge.
```

## Conventions enforced

- All conventions above are derived from this plugin's `CLAUDE.md` and the `ember-octane-fundamentals` / `ember-testing` skills. Do not invent new rules in this command — link the matching skill in the diagnosis.
- Severity is calibrated to the *current* `ember-source` version. The same finding may downgrade from `error` to `warn` on older versions where the pattern is merely deprecated.
- This command is read-only by default. Do not auto-apply patches without an explicit `--fix` flag (not implemented yet).

## See also

- Skill: `ember-octane-fundamentals`
- Skill: `ember-components-and-templates`
- Skill: `ember-services-and-state`
- Skill: `ember-testing`
- Skill: `ember-typescript-and-glint`
- Agent: `ember-architect` for design-level review (not just lint).
