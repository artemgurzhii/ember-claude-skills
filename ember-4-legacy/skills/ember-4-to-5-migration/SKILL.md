---
name: ember-4-to-5-migration
description: The 4.12 LTS → 5.x migration path — what's removed (ember-cli-typescript as a separate addon, RSVP global, classic test API holdouts), the ember-data → @ember-data/request transition (and the WarpDrive direction), Glint adoption with template-imports, optional .gjs/.gts adoption. Use when planning, scoping, or executing a 4 → 5 upgrade.
type: reference
---

# Ember 4.12 → 5.x Migration

If the prerequisites in [`ember-4-recommendations`](../ember-4-recommendations) are met (Embroider at least in compat mode, official TS setup, Octane settled, addons 5-compatible), the 4 → 5 jump is mostly a **version bump plus opt-in features**. The headline work is:

1. The version bump itself.
2. The addons rolling forward.
3. Adopting `<template>` tag and Glint's template-imports environment (optional but recommended).
4. Adopting the `@ember-data/request` request manager (optional).

```
4.12 LTS  →  5.4  →  5.8  →  5.12  →  ...latest LTS
                                         ↓
                                       (continue with `ember/` plugin)
```

## Pre-flight

Verify on 4.12:

- [ ] No `Ember.X` global imports anywhere.
- [ ] All ember-data relationships specify `async` and `inverse`.
- [ ] No classic test API (`moduleForXxx`) anywhere.
- [ ] No `(mut ...)` or `(action ...)` template helpers.
- [ ] Embroider in compat mode (or optimized).
- [ ] If TypeScript: official setup, no `ember-cli-typescript` addon.
- [ ] All addons in `package.json` ship versions that support both 4.12 and 5.x.
- [ ] Test suite green; deprecation workflow file empty or owner-tagged.

If anything's unchecked, finish on 4.12.

## What 5.x removes / changes

| Removed / changed | Replacement |
|---|---|
| `ember-cli-typescript` (the standalone addon) — its functionality is now in the framework | Use `tsconfig.json` directly with `@tsconfig/ember/tsconfig.json`. |
| Implicit injection in some places (`@service` without an explicit name when the resolver is ambiguous) | `@service('foo') declare foo: FooService;` everywhere. |
| Older `ember-data` model registry path | The path moves around in 5.x — use the `ember-data/types/registries/model` augmentation pattern. |
| Some `ember-cli-mirage` 1.x patterns | Bump to 2.x or 3.x depending on the version range; serializer/factory APIs largely unchanged. |
| Several deprecated `RouterService` methods | Use the modern equivalents documented in the upgrade guide. |
| Older Glint environment naming | The `ember-loose` and `ember-template-imports` environments are stable; older configurations need updating. |

The full removed list is in the official [Deprecations app](https://deprecations.emberjs.com); filter to "since: 4.x".

## What 5.x adds

| New | What it gives you |
|---|---|
| `<template>` tag in `.gjs`/`.gts` | Single-file components with strict-mode templates. Replaces `.ts` + `.hbs` pairs over time. |
| Glint `ember-template-imports` environment | Type-checking for `<template>` blocks. Materially better than Glint's loose mode. |
| `@ember-data/request` request manager | Builder-based fetch layer. Adapters/serializers become optional. |
| WarpDrive direction | The next-generation ember-data branding/architecture. Backward-compatible during 5.x. |
| Vite-based dev server (via `@embroider/vite` or `ember-cli-vite`) | Faster HMR. Optional. |
| Polaris RFCs landing in feature flags | Route templates as components, etc. — opt-in via flags during 5.x. |

## Hop-by-hop

5.x's release cadence has been less predictable than 4.x's, but the LTS-by-LTS approach is still the safest. Bump to each LTS in order, run the suite, run the codemods, smoke test, ship. Rough cadence:

### 4.12 → 5.4 LTS

The biggest hop. Most removals land here. Specifically:

- `ember-cli-typescript` is no longer needed — remove it from `package.json`, replace its `tsconfig.json` with `@tsconfig/ember/tsconfig.json`.
- Older ember-data paths get cleaned up — relationships without explicit options error out (they were warnings in 4.12).
- Many addons release "5.x-compatible" majors here — bump them in the same window.
- Embroider becomes the preferred build pipeline for 5.x. If you're still on classic ember-cli, expect rough edges.

```bash
pnpm exec ember-cli-update --to 5.4
pnpm exec ember-cli-update --run-codemods
```

### 5.4 → 5.8 LTS

Quieter. Mostly addon ecosystem catching up.

This is a good moment to:

- Adopt `<template>` tag for a small number of new components. Not a migration of existing code — just new files.
- Install `@glint/environment-ember-template-imports` and turn it on in `tsconfig.json`.

### 5.8 → 5.12 LTS

The last 5.x LTS in the predictable cadence (subject to the actual release schedule). At this point:

- Glint is stable enough for whole-codebase use.
- Most major addons ship Glint signatures.
- The `<template>` tag is everywhere in new code.
- The `@ember-data/request` request manager is the recommended way to do fetches.

This is the **handoff point** to the [`ember/`](../../../ember) plugin's `ember-polaris-migration` skill — Polaris-style authoring is now mainstream.

## TypeScript / Glint adoption order

If you're adding template type-checking during the 5.x window:

1. **Bump to 5.x** with the JS-only TS setup intact.
2. **Stabilize.** Run the suite. Smoke-test.
3. **Install Glint** and the `ember-loose` environment.
4. **Add component signatures** to every component. Most should already have them from the 4.x phase.
5. **Run `glint --watch`** until the codebase is clean.
6. **Add `glint --noEmit` to CI.**
7. **Install `ember-template-imports`** and the matching Glint environment.
8. **Adopt `<template>` tag for new components.** Don't migrate existing ones in the same window.
9. **Plan a slow migration** of existing components to `<template>` tag, opportunistically when you'd touch them anyway.

## ember-data — the request-manager transition

`@ember-data/request` is opt-in throughout 5.x. The transition shape:

```ts
// existing 4.x style — still works in 5.x
const post = await this.store.findRecord('post', '42');

// new 5.x style — coexists with the above
import { findRecord } from '@ember-data/json-api/request';
const { content: post } = await this.store.request(findRecord('post', '42'));
```

Don't rewrite all existing fetches at once. Two practical lines:

1. **New endpoints use the request manager.** Especially custom RPC endpoints — no more custom adapter required.
2. **Existing endpoints stay on the legacy `findRecord`/`query` API** until you're touching them anyway.

The legacy adapter/serializer pipeline survives in 5.x. WarpDrive is the long-term direction; for a 5.x app today, you're fine with both styles coexisting.

## `<template>` tag adoption

Same advice as for the request manager: **new files in `.gts`, existing files stay**. Plan a slow migration.

When you do migrate a file:

1. Rename `foo.ts` + `foo.hbs` → `foo.gts`.
2. Wrap the `.hbs` body in a `<template>...</template>` inside the class.
3. Add `import` statements for every component, helper, and modifier referenced.
4. Run Glint. Fix the resulting errors.
5. Delete the old `.hbs` (it's gone — confirm via `git status`).

See the [`ember/skills/ember-polaris-migration`](../../../ember/skills/ember-polaris-migration) skill for the full workflow.

## Addon survival list (4 → 5)

Rough categorization at the time of 5.x's mature releases:

| Addon | 5.x status |
|---|---|
| `ember-cli-mirage` | Alive. Major bumps but API stable. |
| `ember-power-select` | Alive. Glint signatures shipping. |
| `ember-cli-page-object` | Alive. |
| `ember-test-selectors` | Alive. |
| `ember-modifier` | Alive. Stable API. |
| `@ember/render-modifiers` | Alive. |
| `ember-truth-helpers` | Replaced/wrapped by `@ember/truth-helpers` (built-in). |
| `ember-svg-jar` | Alive. |
| `ember-intl` | Alive. |
| `ember-simple-auth` | Alive. |
| `ember-concurrency` | Alive. Glint signatures shipping. |
| `ember-resources` | Alive. The canonical "resource" pattern. |
| `tracked-built-ins` | Alive. |
| `ember-cli-typescript` (the addon) | Removed/no longer needed. |
| Addons stuck on 4-only with no 5-compat statement | Replace or fork. |

## Time estimates (rough, mid-size app, 4.12 prerequisites met)

| Hop | Calendar days |
|---|---|
| 4.12 → 5.4 | 5–10 |
| 5.4 → 5.8 | 1–3 |
| 5.8 → 5.12 | 2–5 |
| (Optional) Glint adoption | 1–2 weeks separate |
| (Optional) `<template>` tag adoption | Continuous, opportunistic |
| **Total core upgrade** | **~2–3 weeks** |

If 4.x prerequisites *aren't* met, add 2–4 weeks to fix them on 4.12 first.

## Common 4 → 5 surprises

| Surprise | Cause | Fix |
|---|---|---|
| Build fails with "Cannot find module 'ember-cli-typescript'" | Addon removed but a config file still references it. | Delete `ember-cli-typescript` config from `ember-cli-build.js`; remove from `package.json`. |
| `Type 'PostModel' is not assignable to type 'unknown'` | `ModelRegistry` augmentation path changed. | Update the path: `declare module 'ember-data/types/registries/model' { ... }`. |
| Glint reports errors in addons' templates | Addon doesn't ship Glint signatures. | Skip addons in your Glint config, or add a stub registry, or wait for the addon to update. |
| `RouterService.transitionTo` deprecation | Method signature changed. | Use the modern equivalent per the upgrade guide. |
| Some `(component "name" ...)` calls fail | Embroider `staticComponents` requires curried component references. | Replace string with imported component. |
| Test suite hangs after bump | Custom test waiter not migrated to `@ember/test-waiters`. | Migrate the waiter. |

## Verification — at the latest 5.x LTS

- [ ] `ember-source` on the latest 5.x LTS.
- [ ] `ember-data` matching.
- [ ] `ember-cli-typescript` not in `package.json`.
- [ ] All component signatures declared.
- [ ] (Optional) Glint passes in CI on `ember-loose` environment.
- [ ] (Optional) Glint passes on `ember-template-imports` environment.
- [ ] (Optional) New components use `<template>` tag.
- [ ] All addons on 5-compatible versions.
- [ ] Embroider in compat or optimized mode.
- [ ] CI green; suite passes; manual smoke-test passes.
- [ ] Deprecation workflow file empty.

When all green, **switch to the [`ember/`](../../../ember) plugin** for ongoing work — Polaris adoption, the modern addon skill, and the architect/test-engineer agents take over from here.

## See also

- `ember-4-recommendations` — the prerequisite work.
- [`ember/skills/ember-polaris-migration`](../../../ember/skills/ember-polaris-migration) — `<template>` tag adoption.
- [`ember/skills/ember-typescript-and-glint`](../../../ember/skills/ember-typescript-and-glint) — Glint setup at the destination.
- Official: [emberjs.com/releases/](https://emberjs.com/releases/), [Deprecations](https://deprecations.emberjs.com), [`@ember-data/request`](https://api.emberjs.com).
