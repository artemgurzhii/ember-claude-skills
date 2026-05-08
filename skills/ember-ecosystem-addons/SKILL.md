---
name: ember-ecosystem-addons
description: Curated picks from emberobserver.com — auth, async, testing, forms/UI, i18n, observability, build & lint. Use when picking an addon, debugging "what does this addon do," or scaffolding a new feature that should reuse community standards.
type: reference
---

# Ember Ecosystem — Top-Rated Addons

Ember's payoff comes from convention *and* a small set of community addons every serious app uses. Don't reinvent these.

## Catalog

Look up a category, then read the matching reference file. Each one has install commands, canonical usage, and notes on when to reach for it vs alternatives.

| Category | What's in it | Reference |
|---|---|---|
| Authentication | `ember-simple-auth`, `ember-cookies` | [`references/auth.md`](references/auth.md) |
| Async / orchestration | `ember-concurrency` | [`references/async.md`](references/async.md) |
| Testing | `ember-test-selectors`, `ember-cli-page-object`, `ember-cli-mirage`, `ember-a11y-testing`, `@percy/ember` | [`references/testing.md`](references/testing.md) |
| Forms & UI | `ember-basic-dropdown`, `ember-power-{select,calendar,datepicker}`, `ember-headless-form`, `ember-changeset(-validations)`, `ember-primitives`, `ember-truth-helpers`, `@ember/render-modifiers`, `ember-modifier`, `ember-resources`, `ember-cli-flash`, `ember-shepherd`, `ember-page-title`, `ember-svg-jar` | [`references/forms-and-ui.md`](references/forms-and-ui.md) |
| Internationalization | `ember-intl` | [`references/i18n.md`](references/i18n.md) |
| Observability | `@sentry/ember` | [`references/observability.md`](references/observability.md) |
| Build & lint | `ember-template-lint`, `@embroider/*`, utility addons, dev tooling | [`references/build-and-lint.md`](references/build-and-lint.md) |

## Choosing well — heuristics

1. **Check emberobserver.com**: install count, score, last release, TypeScript support. Sub-1k installs and stale releases are red flags.
2. **Prefer Mainmatter / NullVoxPopuli / ember-power-addons / ember-modifier / ember-intl–maintained addons** — these orgs ship reliably.
3. **Beware "Ember 1.x rewrites"** — anything written before Octane often relies on classic patterns.
4. **Read the README + look at the test suite** before installing. Quality of tests = quality of addon.
5. **Don't pile on convenience addons** — every dep is upgrade weight. If the addon wraps 30 lines, copy the 30 lines.

## When NOT to install an addon

- For one-off DOM glue, write a custom modifier (3 lines).
- For one-off transformations, write a helper module.
- For state with no async, just write a class.
- For private API wrappers around `getOwner`, just call `getOwner` once.

## Verification when adopting a new addon

- [ ] Score and install count on emberobserver.com are healthy.
- [ ] Last release is within ~12 months (or there's a maintained fork).
- [ ] The addon has TypeScript types or Glint signatures.
- [ ] You've read the README; you know what one feature you need.
- [ ] You've added it to `package.json` (not duplicated by a transitive dep).

## See also

- `ember-octane-fundamentals` — the patterns these addons assume.
- `ember-testing` — Mirage / page-object / test-selectors integration.
- `ember-services-and-state` — `ember-concurrency` orchestration patterns.
- `ember-typescript-and-glint` — typing addons that ship Glint signatures.
