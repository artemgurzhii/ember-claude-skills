---
name: ember-3-to-4-migration
description: The 3.28 LTS → 4.12 LTS migration path — what's removed (jQuery integration, classic components, Ember.X globals), the ember-data 4 typing rewrite, the optional Embroider adoption, addon survival, and the LTS-by-LTS cadence (3.28 → 4.4 → 4.8 → 4.12). Use when planning, scoping, or executing a 3 → 4 upgrade.
type: reference
---

# Ember 3.28 → 4.12 LTS Migration

3 → 4 is the **enforcement major**: Octane is no longer optional, jQuery is removed, classic components and observers are gone for good. If you finished `ember-3-octane-adoption`, this jump is mostly mechanical — you're trading deprecations for build errors.

If you didn't finish Octane on 3.x, **stop and finish it first**. Bumping to 4.x with classic code remaining will make every screen break at once.

```
3.28 LTS  →  4.4 LTS  →  4.8 LTS  →  4.12 LTS
                                       ↓
                         (continue with ember-4-legacy plugin)
```

## Pre-flight checklist

Before bumping `ember-source` to 4.x, verify on 3.28:

- [ ] No `Ember.Component.extend(...)` anywhere.
- [ ] No `actions: { ... }` hashes.
- [ ] No `(mut ...)` in templates.
- [ ] No `this.$()` (jQuery) in product code or tests.
- [ ] No `Ember.X` global imports (use `@ember/...` modules).
- [ ] No mixins.
- [ ] No observers (or all silenced via deprecation workflow with explicit ownership).
- [ ] All ember-data relationships have explicit `inverse` and `async` flags.
- [ ] All addons have a known-4-compatible version.
- [ ] Tests on modern API (no `moduleForXxx`).
- [ ] CI green; deprecation workflow file empty or owner-tagged.

If any of these are unchecked, fix them on 3.28 before bumping.

## What 4.x removes

A non-exhaustive list. Each line is something that **will fail at build or runtime** in 4.x:

| Removal | What to do |
|---|---|
| jQuery integration (final removal, not just deprecation) | Replace any remaining `this.$()` with native DOM; remove `@ember/optional-features` flag. |
| `Ember.Component` (classic) | Convert to `@glimmer/component`. |
| `Ember.Object.extend({...})` | Convert to native classes. |
| `Ember.Mixin` | Service-ify or extract. |
| `actions: {...}` hash on classic components | Already gone if Octane finished. |
| `Ember.computed` (the global) | Use `import { computed } from '@ember/object'` if absolutely needed; prefer getters with `@tracked` reads. |
| Observers via `Ember.observer` | Convert to actions, derived state, or modifiers. |
| `Ember.run` (the global) | Use `import { run, scheduleOnce, ... } from '@ember/runloop'`. |
| `Ember.RSVP` (the global) | Use `import RSVP from 'rsvp'` directly, or native promises. |
| `Ember.String.htmlSafe` etc. | Use `htmlSafe` from `@ember/template`. |
| `Ember.copy`, `Ember.merge`, `Ember.assign` | Native `structuredClone` / spread / `Object.assign`. |
| `Ember.testing` | Use `import { isTesting } from '@embroider/macros'` if needed. |
| `partial` template helper | Inline the template or extract to a component. |
| `with` template helper | Use `let` instead. |
| Classic test API (`moduleForComponent`, etc.) | Already gone if migrated. |

## Hop-by-hop

### 3.28 → 4.4 LTS

This is the **biggest hop**. Most removals land here.

```bash
pnpm exec ember-cli-update --to 4.4
pnpm exec ember-cli-update --run-codemods
```

Expect:
- Build errors for any remaining classic patterns.
- Errors for any addon that hasn't published a 4-compat release.
- ember-data 4.x: the `Model` typing changed. Many `attr`/`belongsTo`/`hasMany` declarations want explicit options.

Specific to ember-data 4:
- `import DS from 'ember-data'` is gone. Use `import Model, { attr, belongsTo, hasMany } from '@ember-data/model';`.
- `import Adapter from 'ember-data/adapter';` becomes `import Adapter from '@ember-data/adapter';`.
- `import Serializer from 'ember-data/serializer';` becomes `import Serializer from '@ember-data/serializer';`.
- `relationship.async` and `relationship.inverse` must be explicit on every `belongsTo`/`hasMany` (warnings in 3.x, errors in 4.x).

### 4.4 → 4.8 LTS

Quieter hop. Mostly cleanup of new deprecations introduced in 4.4 around:

- `ember-data` request-internals refactor (preparation for the request manager).
- `RouterService` API tightening.
- Implicit injection deprecation (`@service` must name the property explicitly in some setups).

### 4.8 → 4.12 LTS

The final 4.x. Embroider becomes much easier to adopt here.

What lands here for many teams:

- **Embroider opt-in.** If you didn't adopt it before, 4.8-4.12 is the smoothest moment. Enables tree-shaking, ES modules, and prepares the way for `.gjs`/`.gts` in 5.x.
- **TypeScript becomes officially supported** (without needing `ember-cli-typescript` as a separate addon).
- **`@ember/test-waiters`** becomes the canonical way to register async work for the test runtime.

## The Embroider question

Embroider is a different build pipeline. You can adopt it in 3.x, 4.x, or 5.x — it's orthogonal to the Ember version. **But**:

- Polaris (`<template>` tag, `.gjs`/`.gts`) requires Embroider.
- TypeScript with Glint works best on Embroider.
- Many newer addons assume Embroider compatibility.

Recommended sequence:

1. Get to 4.4 on the classic build pipeline.
2. Stabilize.
3. Switch to `@embroider/core` + `@embroider/compat` (the "compat" mode mimics classic resolver behavior).
4. Stabilize again.
5. Move to 4.8 / 4.12.
6. Then transition `compat` → `optimized` mode (real ES modules) at your own pace.

Adopting Embroider is a separate week of work. Don't bundle it with the version bump.

## Addon survival list (3 → 4)

| Addon | Status at 4.x | Notes |
|---|---|---|
| `ember-cli-mirage` | Alive. | Bump to 2.x or 3.x; API is largely stable. |
| `ember-power-select` | Alive. | Bump major. |
| `ember-cli-page-object` | Alive. | Keep. |
| `ember-test-selectors` | Alive. | Keep. |
| `ember-modifier` | Alive. | API rewrite around 4.x — re-author custom modifiers if you have many. |
| `@ember/render-modifiers` | Alive. | Keep. |
| `ember-truth-helpers` | Alive. | Eventually replaced by `@ember/truth-helpers`. |
| `ember-svg-jar` | Alive. | Keep. |
| `ember-intl` | Alive. | Bump major; the API became more closely aligned with Intl. |
| `ember-simple-auth` | Alive (Mainmatter). | Bump major. |
| `ember-concurrency` | Alive. | Bump major; signature types added. |
| `ember-cli-tailwind` | Less maintained. | Migrate to direct Tailwind via PostCSS. |
| `liquid-fire` | Likely abandoned. | Replace with `ember-animated` or hand-rolled. |
| Any addon < 1k installs and last release > 3y | Replace or fork. | Migrating an abandoned addon is rarely worth it. |

## Tests during the jump

- Run the suite at every hop. Smoke-test the top user flows manually after each.
- **Component test infrastructure changes**: `@ember/test-helpers` major bump can shift behavior of `settled()` and `waitFor`. Check the changelog.
- **Mirage 2 → 3**: small API differences, mostly around model/serializer relationships. Run all integration tests.

## Time estimates (rough, mid-size app, Octane finished)

| Hop | Calendar days |
|---|---|
| 3.28 → 4.4 | 5–10 |
| 4.4 → 4.8 | 1–3 |
| 4.8 → 4.12 | 2–5 (more if adopting Embroider/TS) |
| **Total** | **~2–3 weeks** |

If Octane wasn't finished on 3.x, **add 2–4 weeks** for the cleanup that should have happened before the bump.

## Common 3 → 4 surprises

| Surprise | Cause | Fix |
|---|---|---|
| Whole pages render blank after the bump | Classic component still in the layout — its wrapper element disappeared. | Convert the layout component to `@glimmer/component` and use `...attributes`. |
| `belongsTo`/`hasMany` returns `null` unexpectedly | Missing `inverse`. | Specify `inverse: 'fieldName'` on both sides. |
| Test suite hangs after bump | A custom waiter wasn't re-registered for the new `@ember/test-helpers` major. | Re-register or migrate to `@ember/test-waiters`. |
| `Cannot read properties of undefined (reading 'extend')` in an addon | Addon imports `Ember.Component`. | Bump the addon, or fork it. |
| Build is much slower after Embroider | `@embroider/compat` mode in dev, full reanalysis on every change. | Tune `staticAddonTrees: true` once stable; consider `optimized` mode in CI. |
| `htmlSafe` undefined | Imported from `Ember.String`. | `import { htmlSafe } from '@ember/template';` |

## Verification — at the 4.12 finish line

- [ ] `ember-source` is `~4.12.x`.
- [ ] `ember-data` is on the matching 4.x major.
- [ ] No `@ember/optional-features` flag for `jquery-integration` (it's not a real flag anymore).
- [ ] Build passes without warnings.
- [ ] All addons reported by `ember-cli-update --addons` are on 4-compatible versions.
- [ ] Tests green; deprecation workflow file empty or near-empty.
- [ ] (Optional) Embroider in compat or optimized mode.
- [ ] (Optional) TypeScript via the official setup.
- [ ] CI is green.

When the box is checked, switch to the [`ember-4-legacy`](../../../ember-4-legacy) plugin's `ember-4-to-5-migration` skill.

## See also

- `ember-3-octane-adoption` — the prerequisite.
- `ember-4-legacy/skills/ember-4-to-5-migration` — the next hop.
- Official: [emberjs.com/releases/](https://emberjs.com/releases/), [Deprecations app](https://deprecations.emberjs.com), [Embroider docs](https://github.com/embroider-build/embroider).
