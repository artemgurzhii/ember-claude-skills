# Build, lint & utility addons

## Linting / quality

### [`ember-template-lint`](https://github.com/ember-template-lint/ember-template-lint)

The linter for `.hbs` and `.gjs` templates. Comes with sensible defaults (`recommended` and `octane` configs). Catches `<input @value=...>` mistakes, missing `type=` on buttons, accessibility issues (`a11y-accessibility` rules), and Octane anti-patterns.

```bash
pnpm add -D ember-template-lint
pnpm exec ember-template-lint .
```

### [`@embroider/*`](https://github.com/embroider-build/embroider)

Embroider is the modern build pipeline for Ember — it produces standard ES modules, supports tree-shaking, and is the foundation for Vite-based dev servers (`ember-cli-vite`). Most modern apps run Embroider in "Optimized" mode.

```bash
ember install @embroider/core @embroider/compat @embroider/webpack
```

Polaris assumes Embroider.

## Utility addons worth knowing

| Addon | What it gives you |
|---|---|
| [`tracked-built-ins`](https://github.com/tracked-tools/tracked-built-ins) | `TrackedArray`, `TrackedMap`, `TrackedSet`, `TrackedWeakMap` — reactive primitives with a familiar API. |
| [`ember-element-helper`](https://github.com/tildeio/ember-element-helper) | `(element "h1")` to render dynamic tags. |
| [`ember-keyboard`](https://github.com/adopted-ember-addons/ember-keyboard) | Declarative keyboard shortcuts with priority/scope. |
| [`ember-animated`](http://ember-animated.com) | Animations and transitions tied to Ember's render cycle. |
| [`ember-cli-deprecation-workflow`](https://github.com/mixonic/ember-cli-deprecation-workflow) | Bulk-snooze deprecations during upgrades, fix them in waves. |

## Dev tooling

| Tool | Purpose |
|---|---|
| [Ember Inspector](https://github.com/emberjs/ember-inspector) (browser ext) | Routes, components, services, store records — your X-ray. |
| [`ember-auto-import`](https://github.com/embroider-build/ember-auto-import) | `import 'lodash'` from npm without an addon wrapper. (Default in modern setups.) |
