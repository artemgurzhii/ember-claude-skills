---
name: ember-3-recommendations
description: Practical advice for teams settling on Ember 3.28 LTS as a checkpoint — when to keep going to 4.x, when to pause, what to do during the pause. Use when triaging a 3.28 codebase, deciding whether to invest in Octane within 3.x first, or planning a quarterly upgrade roadmap.
type: feedback
---

# Ember 3.x — Recommendations

3.28 is the longest-lived 3.x LTS and a defensible **checkpoint** in a long upgrade journey. It is **not** a defensible long-term home. Security patches for 3.x stopped in 2023; running 3.28 in 2026 is in the same risk class as running 2.x in 2020.

## When to pause at 3.28 vs press on

| Situation | Recommendation |
|---|---|
| 3.28, classic syntax everywhere, thin tests | **Pause.** Adopt Octane within 3.x first (see `ember-3-octane-adoption`). Then proceed to 4.x. |
| 3.28, fully Octane, strong tests, jQuery removed | **Press on.** 3 → 4 is one of the easier jumps. Don't linger here. |
| 3.16-3.27, mid-Octane | **Keep moving** to 3.28 first; the deprecation messages in 3.20+ are clearer signal than 3.16's. |
| 3.4 LTS or earlier 3.x | Catch up to 3.28 before doing anything else. |

The shape of the code matters more than the version number. **Octane shape on 3.28** is a much better starting point for the 4.x jump than **classic shape on 3.28**.

## Order of operations during the pause

If you're going to sit on 3.28 for a quarter or two while finishing Octane adoption:

1. **Lock the version.** Pin `ember-source@~3.28.x`, `ember-cli@~3.28.x`, `ember-data@~3.28.x`. Lockfile committed.
2. **Empty the deprecation workflow.** Every silenced deprecation gets either a fix or an issue with an owner.
3. **Adopt Octane** (see the dedicated skill).
4. **Convert tests** to the modern API where possible.
5. **Add `data-test-*` selectors** everywhere.
6. **Audit addons.** Identify the ones that won't survive 4.x (no recent release, no Ember 4 compatibility). Replace or fork.
7. **Embroider opt-in** *after* steps 1–6. `@embroider/core` + `@embroider/compat` + `@embroider/webpack`. This is a separate project; it's the hardest single change in the migration arc. Many teams skip Embroider on 3.x and adopt it during 4.x.

## Octane-on-3.28 — why it's worth doing here, not later

A common mistake: "Let's just bump to 4.x, then adopt Octane." Don't. 4.x **requires** Octane. If you bump first, you discover every classic pattern at once, while also fighting major-version removals and an ember-data 4.x typing rewrite. Conversely, doing Octane on 3.28:

- The classic API is still present as a fallback. Each component you convert is opt-in, not forced.
- Deprecation messages are clearer — they tell you exactly which Octane pattern to adopt.
- The codemods are tuned for this exact transition.
- The test infrastructure works in both APIs simultaneously.

Most teams that struggle with the 4.x jump are teams that didn't finish Octane in 3.x.

## Addon hygiene at 3.28

Things to actively check:

- **Is the addon still maintained?** Last release date < 18 months → safe. > 3 years → likely dead.
- **Does it ship Glint signatures?** If you'll add TS in 4.x, addons without signatures become friction.
- **Does it have a 4.x compat statement?** Look at the README and recent issues.
- **Does it have a non-classic alternative?** `ember-i18n` is dead; use `ember-intl`. Old date pickers may have current alternatives.

A practical script: list addons in `package.json`, look each up on emberobserver.com, decide keep/replace/fork. Bake the decision into a `MIGRATION.md` in the repo.

## What to keep doing

- **Mirage** scenarios — they survive every upgrade.
- **`ember-test-selectors`** — keeps tests independent of design.
- **`ember-cli-page-object`** — fine through every version.
- **`ember-power-select`** — keep current major.
- **`ember-cli-deprecation-workflow`** — your friend during *every* upgrade.

## What to stop doing

- **Don't write classic components.** Even a one-line tweak is a chance to convert.
- **Don't add new mixins.** Service or utility instead.
- **Don't write classic-style tests** (`moduleForComponent`). All new tests on the modern API.
- **Don't introduce TypeScript.** Save it for 4.x or 5.x — type infrastructure improved dramatically there. (If you must, use the official `ember-cli-typescript`-era setup, but expect to redo it later.)

## CI gates worth adding at 3.28

- `template-lint --config octane` blocks classic-only patterns.
- `eslint-plugin-ember`'s `octane` ruleset blocks `Ember.Component.extend`, `actions: {}`, etc.
- A check that fails on new entries in `deprecation-workflow.js` (the file should shrink or stay the same, never grow).
- `ember-template-lint` deny-list rules for `(mut ...)` in templates (or upgrade-block rule).

## Backup plan

Before you start the 4.x bump:

1. Make a `release/3.28` branch from your latest green main.
2. Tag `v-pre-4-upgrade`.
3. Set up CI to allow hot-fix releases off `release/3.28` for the duration of the migration.

If something explodes during the 4.x window, you can ship a 3.28 fix without unwinding the migration.

## Verification — ready for 4.x

- [ ] On Ember 3.28.x, with locked deps.
- [ ] Octane optional features all enabled.
- [ ] No classic components in `app/components/`.
- [ ] No `(mut ...)` in templates.
- [ ] No `Ember.X` global imports.
- [ ] No `this.$()` in product code.
- [ ] All addons have a 4.x-compatible version pinned in `package.json` (i.e., no addon will block a 4.x bump).
- [ ] Tests are on `module + setupApplicationTest/setupRenderingTest`.
- [ ] CI gates against new classic patterns.
- [ ] `release/3.28` branch exists for emergency fixes.

When that's all checked, switch to `ember-3-to-4-migration` and start the bump.

## See also

- `ember-3-octane-adoption` — the prerequisite for a clean 4.x jump.
- `ember-3-to-4-migration` — the next step.
- `ember-2-recommendations` — analogous skill for the previous version.
