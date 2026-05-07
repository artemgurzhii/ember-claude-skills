# ember

A Claude Code plugin for working in EmberJS codebases — focused on **Octane** today and the **Polaris** transition (template tag, strict mode `.gjs` / `.gts`).

## What's inside

### Skills (`skills/`)
| Skill | Purpose |
|---|---|
| `ember-octane-fundamentals` | Native classes, `@tracked`, `@action`, decorators, owner / DI, autotracking model. |
| `ember-components-and-templates` | Glimmer components, args, blocks, `{{yield}}`, modifiers, helpers, splattributes. |
| `ember-routing-and-models` | Router, route hooks (`model`, `beforeModel`, `afterModel`), nested routes, query params, `<LinkTo>`, transitions. |
| `ember-services-and-state` | Singleton services, DI via `@service`, app-wide state, `@cached`, derived state. |
| `ember-data` | Store, models, relationships, adapters, serializers, JSON:API, custom requests, `@ember-data/request`. |
| `ember-testing` | QUnit + `@ember/test-helpers`, application/integration/unit tests, `ember-test-selectors`, `ember-cli-page-object`, `ember-cli-mirage`, a11y. |
| `ember-ecosystem-addons` | Top-rated addons from emberobserver.com — `ember-simple-auth`, `ember-concurrency`, `ember-power-select`, `ember-modifier`, `ember-resources`, `ember-intl`, `ember-svg-jar`, and more. |
| `ember-typescript-and-glint` | TS in Ember, `@glint/environment-ember-loose` vs `@glint/environment-ember-template-imports`, type-safe templates. |
| `ember-polaris-migration` | Template tag (`<template>`), `.gjs`/`.gts`, strict mode, route templates, mental model shifts. |

### Agents (`agents/`)
| Agent | Use for |
|---|---|
| `ember-architect` | Designing Ember apps — feature layout, routing/state shape, addon selection, migration plans. |
| `ember-test-engineer` | Test strategy and authoring — QUnit, page objects, Mirage scenarios, a11y, performance assertions. |

### Commands (`commands/`)
| Command | Action |
|---|---|
| `/ember-component` | Scaffold a Glimmer component with template, backing class, and integration test. |
| `/ember-route` | Scaffold a route with `model` hook, controller (if needed), template, and application test. |
| `/ember-service` | Scaffold a service with TS types and a unit test. |
| `/ember-test` | Run the test suite (QUnit) and triage failures. |

## Install (local)

From the Claude Code CLI:

```
/plugin install /Users/artemhurzhii/Programming/Work/ember-claude-skills/ember
```

## Sources

This plugin draws from:

- The official [Ember Guides](https://guides.emberjs.com) and [API docs](https://api.emberjs.com).
- The [Ember RFCs](https://rfcs.emberjs.com) — especially Octane (RFC #176) and Polaris-track RFCs (#779 template tag, #496 `<template>` tag, #756 strict mode).
- Reference repositories: [ember-source](https://github.com/emberjs/ember.js), [ember-data](https://github.com/emberjs/data), [ember-cli](https://github.com/ember-cli/ember-cli), [glimmer-vm](https://github.com/glimmerjs/glimmer-vm), and the [empress/field-guide](https://github.com/empress) examples.
- Community blueprints from [emberobserver.com](https://emberobserver.com) — addons listed in `skills/ember-ecosystem-addons/SKILL.md`.
