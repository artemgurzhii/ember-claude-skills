---
name: ember-4-recommendations
description: Practical advice for teams settling on Ember 4.12 LTS — when to press on to 5.x, when to pause, what to invest in during the pause (Embroider, addons, TS-without-Glint). Use when triaging a 4.12 codebase, planning quarterly roadmap, or deciding the order of Embroider/TS adoption.
type: feedback
---

# Ember 4.x — Recommendations

4.12 LTS is the last 4.x. It's a defensible **resting point** before the 5.x jump — much more so than 3.28 was for 3 → 4. Octane is settled; the codebase shape is the modern shape; only the build pipeline and template-authoring story (Polaris) change in 5.x.

## When to pause at 4.12 vs press on

| Situation | Recommendation |
|---|---|
| 4.12, classic ember-cli build pipeline, no Embroider | **Pause.** Adopt Embroider compat first. Then 5.x. |
| 4.12, Embroider compat, JS-typed | **Press on.** 5.x is straightforward from this state. |
| 4.12, Embroider optimized | **Press on.** This is a textbook ready state. |
| 4.4 / 4.8 LTS | Bump to 4.12 first. |
| Any 4.x with TS adopted via `ember-cli-typescript` | **Migrate to the official TS setup before 5.x.** Keeping the addon dependency complicates the bump. |

## Pause investments — order of operations

If you're at 4.12 and not bumping immediately, invest in **this order**:

1. **Empty the deprecation workflow.** Every silenced deprecation gets a fix or an owner.
2. **Adopt Embroider compat.** `@embroider/core` + `@embroider/compat` + `@embroider/webpack`. Settle for two weeks.
3. **Audit addons** for 5.x compatibility:
   - Look at each addon's repo for a `5.x` compatibility statement.
   - Pin to the latest version that supports both 4.12 and 5.x.
   - Replace or fork anything stuck on 4-only.
4. **Move from `ember-cli-typescript` (if used) to the official TS setup.**
5. **Type the JS layer** (Registry pattern for services and models, signatures on components).
6. **Move toward Embroider optimized mode** — `staticAddonTrees`, `staticHelpers`, `staticModifiers`, `staticComponents`. This often surfaces dynamic-component-by-string usages that need to be made static.
7. **Stop here. Bump to 5.x.**

If you try to do this in a different order — e.g. adopt Glint before Embroider, or jump to 5.x with the classic build pipeline — every step gets harder.

## Embroider — practical adoption

The shortest-risk path is the official three-step:

```bash
pnpm add -D @embroider/core @embroider/compat @embroider/webpack
```

Replace `ember-cli-build.js`:

```js
'use strict';
const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function (defaults) {
  const app = new EmberApp(defaults, {
    // your existing options
  });

  const { Webpack } = require('@embroider/webpack');
  return require('@embroider/compat').compatBuild(app, Webpack, {
    // start with all flags FALSE (compat mode)
    staticAddonTrees: false,
    staticAddonTestSupportTrees: false,
    staticHelpers: false,
    staticModifiers: false,
    staticComponents: false,
    splitAtRoutes: [],
    packagerOptions: {
      webpackConfig: {}
    }
  });
};
```

Run the suite. Smoke-test. Once stable for a sprint, flip flags one at a time:

1. `staticAddonTrees: true`
2. `staticAddonTestSupportTrees: true`
3. `staticHelpers: true`
4. `staticModifiers: true`
5. `staticComponents: true` — last and hardest. Surfaces dynamic-component-by-string usages.

For each `static*` flag, expect to find a few dynamic-by-string usages that need to be replaced with curried component references:

```hbs
{{!-- before — fails with staticComponents: true --}}
{{component this.componentName @data=foo}}

{{!-- after --}}
{{this.dynamicComponent @data=foo}}
```

Where `this.dynamicComponent` is a `(component MyComponent)` curried reference, *not* a string.

## Addon hygiene at 4.12

Things to actively check:

- **5.x compatibility statement.** Look in the README and `package.json`'s `peerDependencies.ember-source` range.
- **Glint signatures.** If the addon ships them, your eventual TS adoption is much smoother.
- **Embroider compatibility.** Some older addons fail to compile under static modes. Check the addon's GitHub issues.
- **Last release date.** > 12 months stale + heavy use = fork now, don't wait.

A practical script: list addons in `package.json`, check each on emberobserver.com, mark each as **keep / bump / replace / fork** and record the decision in `MIGRATION.md`.

## Things to keep doing

- **Mirage** — works the same.
- **`ember-test-selectors`, `ember-cli-page-object`** — keep.
- **`ember-power-select`** — keep current major.
- **`ember-simple-auth`** — keep current major (Mainmatter actively maintains).
- **`ember-concurrency`** — keep current major.
- **`ember-modifier`** — keep current major.
- **`ember-intl`** — keep current major.

## Things to stop doing on 4.x

- **Don't ship classic-pattern code** (it doesn't compile, but addon deps sometimes sneak it in).
- **Don't write `(action ...)` template helpers** — only `(fn ...)` with method references.
- **Don't introduce dynamic component invocation by string** — Embroider optimized mode forbids it.
- **Don't ignore the `@embroider/macros` package** when an addon assumes it.

## CI gates worth adding at 4.12

- `tsc --noEmit` (if any TS).
- `template-lint --config octane` (or `octane:strict` for stricter rules).
- `eslint-plugin-ember` recommended ruleset.
- Failure on new `Ember.X` imports.
- Bundle-size budget (Embroider tree-shakes; regressions are visible).

## Backup plan before the 5.x bump

1. Tag a `release/4.12` branch.
2. Tag `v-pre-5-upgrade`.
3. Configure CI to allow hotfixes on `release/4.12`.
4. Document the rollback in `MIGRATION.md`.

## When to skip the 5.x bump and rewrite

Same heuristic as in `ember-2-recommendations`. If multiple are true, scope a rewrite into the latest LTS on a separate branch:

- Tests covering < 50% of user flows after the upgrade investments.
- Addon ecosystem heavily forked / bespoke.
- Build pipeline customized in ways Embroider can't absorb.
- App startup > 5s on a fast laptop, with no clear cause.
- Active product development is slowed > 30% by the upgrade burden.

The rewrite path is: build a fresh Ember 5/6 app with the `ember/` plugin's conventions; route-by-route migrate the 4.x app behind a path-prefix proxy; sunset.

## Verification — ready for 5.x

- [ ] On Ember 4.12.x, locked.
- [ ] `ember-data` 4.12.x.
- [ ] `ember-cli-typescript` (the addon) is removed; TS setup is official.
- [ ] All component signatures declared.
- [ ] All services and models registry-augmented.
- [ ] Embroider in **at least compat mode**, ideally optimized.
- [ ] All addons pinned to versions that work on both 4.12 and 5.x.
- [ ] Tests green; deprecation workflow file empty.
- [ ] CI gates against new classic patterns and bundle bloat.
- [ ] `release/4.12` branch tagged.

When all green, switch to `ember-4-to-5-migration`.

## See also

- `ember-4-octane-only` — what's available.
- `ember-4-typescript-early` — typing strategy.
- `ember-4-to-5-migration` — the next jump.
- The modern Ember skills (`ember-octane-fundamentals`, `ember-polaris-migration`, `ember-typescript-and-glint`) — the destination.
