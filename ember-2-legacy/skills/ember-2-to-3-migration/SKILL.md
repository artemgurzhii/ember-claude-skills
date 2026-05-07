---
name: ember-2-to-3-migration
description: The 2.18 LTS → 3.28 LTS migration path — the LTS-by-LTS sequence, ember-cli-update, the deprecation workflow, jQuery removal, the test API conversion, and the addon survival list. Use when planning, scoping, or executing a 2.x upgrade. Pairs with the ember-3-legacy plugin once you reach 3.x.
type: reference
---

# Ember 2.18 → 3.28 LTS Migration

The fastest, safest path from Ember 2.x is to walk the LTS chain. Each LTS is a stable target; each hop fixes the deprecations the next hop expects to be gone.

```
2.18 LTS  ─┐
           ├→  3.4 LTS   ─┐
                          ├→  3.8 LTS  ─┐
                                        ├→  3.12 LTS  ─┐
                                                       ├→  3.16 LTS  ─┐
                                                                      ├→  3.20 LTS  ─┐
                                                                                     ├→  3.24 LTS  ─┐
                                                                                                    ├→  3.28 LTS
                                                                                                       (last 3.x)
```

You do **not** jump directly. The shortest path that works in practice is **2.18 → 3.4 → 3.8 → 3.12 → 3.16 → 3.20 → 3.24 → 3.28**. Each hop is one or two days of work for a typical mid-size app; it can be many weeks for a 2014-era monolith.

Once at 3.28, switch to the [`ember-3-legacy`](../../../ember-3-legacy) plugin to plan 3 → 4.

## Tool of record: `ember-cli-update`

[`ember-cli-update`](https://github.com/ember-cli/ember-cli-update) regenerates the project files for a target Ember/CLI version, then asks you to merge the diff. It's the single best leverage point in the upgrade.

```bash
# install once
pnpm add -D ember-cli-update

# bump to a target version
pnpm exec ember-cli-update --to 3.4

# resolve the conflicts it surfaces, then:
pnpm exec ember-cli-update --run-codemods
```

Codemods you'll meet:

| Codemod | Purpose |
|---|---|
| `ember-modules-codemod` | `Ember.Component` → `import Component from '@ember/component'` (and the rest of the `Ember.X` namespace explosion). |
| `ember-test-helpers-codemod` | 2.x global helpers → `await visit(...)` style. |
| `ember-no-implicit-this-codemod` | `{{foo}}` (when `foo` is a class field) → `{{this.foo}}`. |
| `ember-angle-brackets-codemod` | `{{user-card}}` → `<UserCard>`. |
| `ember-on-modifier-codemod` | `{{action "x"}}` on elements → `{{on "click" this.x}}`. |
| `ember-tracked-properties-codemod` | computed → `@tracked` (run during the Octane phase). |

Codemods are *helpers*, not magic. Always review the diff and run tests.

## The deprecation workflow — the actual loop

The repeated pattern at every hop:

```
1. Update ember-source, ember-cli, ember-data to the next LTS.
2. Boot the app, watch the console.
3. For every deprecation message:
     a. Find the offending line via the stack.
     b. Either fix it or stash it via ember-cli-deprecation-workflow.
4. Run the test suite. Fix breakage.
5. Smoke-test the top user flows manually.
6. Commit per-deprecation when reasonable, so the diff is reviewable.
7. Move to the next LTS.
```

Use [`ember-cli-deprecation-workflow`](https://github.com/mixonic/ember-cli-deprecation-workflow) to **silence deprecations you don't have time to fix yet**:

```js
// config/deprecation-workflow.js
self.deprecationWorkflow = self.deprecationWorkflow || {};
self.deprecationWorkflow.config = {
  workflow: [
    { handler: 'silence', matchId: 'ember-component.send-action' },
    { handler: 'silence', matchId: 'ember-views.curly-components.jquery-element' }
  ]
};
```

This unblocks the hop without committing to fixing every deprecation in the same week. **Track silenced deprecations in an issue** and burn them down later.

## Hop-by-hop guide

### 2.18 → 3.4

This is the deepest hop because it crosses the major version line. The rest are easier.

What changes:
- `Ember.X` global access starts emitting deprecations everywhere → run `ember-modules-codemod`.
- `ember-cli` configuration changes (`ember-cli-build.js` shape, addon hooks).
- Bower is dropped — anything you load via `bower.json` must move to npm.
- `ember-data 3.x` requires explicit relationship inverses to be set if you customize them.
- The 2.x test API still works, but `ember-qunit@^4` adds the new style alongside.

What to watch:
- jQuery is still bundled. You'll see the *deprecation* about `Ember.$` and `this.$()`, but it's not removed yet.
- Closure actions (`(action "save" post)`) replace `sendAction` everywhere.

### 3.4 → 3.8

- `componentName.toString()` deprecations.
- More `_super` chain warnings.
- Stricter `RouterDSL` (e.g. nested route paths must not start with `/`).

### 3.8 → 3.12

- **Octane appears in the air**: `@glimmer/component`, `@tracked`, modifiers, native classes, decorators are all stable but optional. Don't adopt yet — finish the deprecation pass first.
- jQuery becomes optional (`@ember/optional-features`). You can disable it now — many tests will need updating from `this.$()` to `find()`.

```bash
pnpm exec ember feature:disable jquery-integration
```

### 3.12 → 3.16

- `Ember.run` namespace deprecations: rename to `@ember/runloop` imports.
- Deprecations around `Ember.String` extensions (`'foo'.camelize()` etc.) — import explicitly from `@ember/string`.
- Glimmer 2 compilation pipeline becomes the default. Some HTMLBars-only tricks stop working.

### 3.16 → 3.20

- Many more "Octane mental model" hints in the console. Still optional, but the runway is shrinking.
- ember-cli reaches a build pipeline that handles ES modules cleanly — many addon issues evaporate at this point.

### 3.20 → 3.24

- Component naming becomes stricter (the resolver expects file paths to match invocation names).
- `tagless` components emit clearer deprecations.

### 3.24 → 3.28

The final 3.x. **3.28 LTS** is the canonical "stop here, take stock, plan the 4.x jump" station.

What to do at 3.28:
- All `Ember.*` global access is gone (or silenced).
- Tests are mostly modernized (`module + setupApplicationTest + await`).
- Octane is opted-in via `config/optional-features.json`:
  ```json
  {
    "application-template-wrapper": false,
    "default-async-observers": true,
    "jquery-integration": false,
    "template-only-glimmer-components": true
  }
  ```
- Components have been **selectively** ported to `@glimmer/component` — usually the leaf components first.
- The deprecation workflow file should be empty or near-empty.

This is a great moment to add CI gates: error on new deprecations, fail on new `Ember.*` imports, lint rules against `extend()`-style classes.

## jQuery removal — usually the longest pole

jQuery removal cuts across templates, components, and tests. A practical order:

1. **Tests first.** Replace `this.$(...)` with `find(...)`, `findAll(...)`, `assert.dom(...)`. Smaller blast radius.
2. **Component DOM hooks.** `didInsertElement(){ this.$('.x').focus(); }` → modifier (`{{auto-focus}}`).
3. **AJAX.** `Ember.$.ajax(...)` → `fetch` or `ember-fetch`. Keep response shape identical to avoid serializer changes.
4. **Plugins.** Any "wrap a jQuery plugin" pattern → custom modifier with proper teardown.

Disable the integration with `ember feature:disable jquery-integration` only **after** the four steps above are done. If you turn it off prematurely, every render will break loudly.

## Ember Data hop

`ember-data 2.x → 3.x → 3.28` is a parallel track:

- `DS.Model.extend({...})` still works in 3.x but begins emitting deprecations late in 3.x.
- Implicit relationship inverses become explicit-required. Fix `@belongsTo`/`@hasMany` to specify `inverse: 'fieldName'` at every site.
- `import DS from 'ember-data'` is gradually replaced by `import Model, { attr, belongsTo, hasMany } from '@ember-data/model'`.
- The `RESTSerializer`/`JSONAPISerializer` defaults stayed stable; *don't* rewrite them mid-migration.

## Addon survival list

Common 2.x addons and their fate:

| Addon | Status | Action |
|---|---|---|
| `ember-cli-mirage` | Alive, well-maintained. | Keep. |
| `ember-power-select` | Alive. | Bump to current major. |
| `ember-cli-page-object` | Alive. | Keep. |
| `ember-test-selectors` | Alive. | Keep. |
| `ember-cli-deprecation-workflow` | Alive. | **Use it during the migration**, remove after 3.28. |
| `ember-truth-helpers` | Alive (now `@ember/truth-helpers` in 5+). | Keep, rename later. |
| `ember-svg-jar` | Alive. | Keep. |
| `ember-i18n` | Replaced by `ember-intl`. | Migrate translations. |
| `ember-data-table-of-the-week` (any 2.x table addon) | Almost certainly dead. | Replace with a hand-rolled table component or a current addon. |
| `liquid-fire` | Lower maintenance. | If used heavily, evaluate `ember-animated`; otherwise delete. |
| Any addon last released > 4 years ago and used in `< 3` files | Inline the code or replace. | Don't try to upgrade an abandoned addon. |

Run `pnpm exec ember-cli-update --addons` (in newer versions of the tool) to surface addons that block the bump.

## Tests — the safety net throughout

Before each hop:

1. Run the full suite — baseline green.
2. Bump.
3. Run the full suite — fix until green again.
4. **Manually smoke-test** the top user flows. Tests don't catch everything; in 2.x they often miss a lot.

If your suite is thin, the first investment is more tests, not the upgrade. Migrating a flaky test suite is *worse* than not migrating.

## Time estimates (rough, mid-size app)

| Hop | Calendar days | Notes |
|---|---|---|
| 2.18 → 3.4 | 5–10 | Heaviest. `Ember.X` rewrite + bower removal. |
| 3.4 → 3.8 | 1–3 | Mostly deprecation cleanup. |
| 3.8 → 3.12 | 2–5 | jQuery starts coming out. Optional, but worth it. |
| 3.12 → 3.16 | 1–3 | Run-loop and string-extension imports. |
| 3.16 → 3.20 | 1–2 | Quiet hop. |
| 3.20 → 3.24 | 1–2 | Quiet hop. |
| 3.24 → 3.28 | 2–5 | Set up Octane optional features; convert leaf components. |
| **Total** | **~3–5 calendar weeks** | For an experienced engineer working full-time. |

Add a multiplier for:
- Heavy classic-component DOM manipulation.
- Many archived addons (each one needs an own decision).
- Thin tests (you'll be hand-verifying flows).

## Verification — at the 3.28 finish line

- [ ] `ember-source` is `~3.28.x`.
- [ ] `ember-cli-update` reports no pending diffs for 3.28.
- [ ] All `Ember.X` direct imports have been replaced or silenced.
- [ ] `jquery-integration` optional feature is `false`.
- [ ] Tests run on the modern API (`module + setupApplicationTest`); `andThen` is gone.
- [ ] `ember-cli-deprecation-workflow.js` is empty or has only well-known, owner-assigned silences.
- [ ] Octane optional features are enabled in `config/optional-features.json`.
- [ ] Addons are upgraded to versions that work on 3.28.
- [ ] `pnpm test` is green; CI is green.

When the box is fully checked, switch to the [`ember-3-legacy`](../../../ember-3-legacy) plugin's `ember-3-to-4-migration` skill.

## See also

- `ember-2-recommendations` — surviving while you plan.
- `ember-3-legacy/skills/ember-3-to-4-migration` — the next hop.
- Official: [emberjs.com/releases](https://emberjs.com/releases/), [Deprecations app](https://deprecations.emberjs.com), [`ember-cli-update`](https://github.com/ember-cli/ember-cli-update).
