# EmberJS Claude skills

A single Claude Code plugin covering **every Ember era from 2.x to the latest 5.x (Octane + Polaris)**. Maintained by [@artemhurzhii](https://github.com/artemhurzhii).

The plugin auto-routes by description: a session in a 5.x app pulls in modern skills; a session in a 2.18 app pulls in `ember-2-classic-patterns`. Mid-migration sessions naturally see both sides at once.

## Install

From a Claude Code session:

```
/plugin marketplace add artemgurzhii/ember-claude-skills
/plugin install ember-claude-skills@artemhurzhii
```

## What's inside

### Skills

**Modern (Ember 5.x+ / Octane + Polaris)**

| Skill | Purpose |
|---|---|
| `ember-octane-fundamentals` | Native classes, `@tracked`, `@action`, decorators, owner / DI, autotracking model. |
| `ember-components-and-templates` | Glimmer components, args, blocks, `{{yield}}`, modifiers, helpers, splattributes. |
| `ember-routing-and-models` | Router, route hooks, nested routes, query params, `<LinkTo>`, transitions. |
| `ember-services-and-state` | Singleton services, DI via `@service`, app-wide state, `@cached`, derived state. |
| `ember-data` | Store, models, relationships, adapters, serializers, JSON:API, `@ember-data/request`. |
| `ember-testing` | QUnit + `@ember/test-helpers`, `ember-test-selectors`, `ember-cli-page-object`, `ember-cli-mirage`, a11y. |
| `ember-ecosystem-addons` | Top-rated addons from emberobserver.com — `ember-simple-auth`, `ember-concurrency`, `ember-power-select`, `ember-modifier`, `ember-resources`, `ember-intl`, `ember-svg-jar`, and more. |
| `ember-typescript-and-glint` | TS in Ember, `@glint/environment-ember-loose` vs `@glint/environment-ember-template-imports`, type-safe templates. |
| `ember-polaris-migration` | Template tag (`<template>`), `.gjs`/`.gts`, strict mode, route templates, mental model shifts. |

**Ember 4.x (Jan 2022 – 4.12 LTS Aug 2023) — Octane-only era**

| Skill | Purpose |
|---|---|
| `ember-4-octane-only` | What 4.x removed — no classic anything, jQuery gone, `Ember.X` gone, observers gone. |
| `ember-4-typescript-early` | Adopting TypeScript on 4.x without Glint, the Registry pattern, when `ember-cli-typescript` fits. |
| `ember-4-recommendations` | Picking 4.12 as a checkpoint, Embroider adoption, addon hygiene. |
| `ember-4-to-5-migration` | The path **4.12 → 5.x latest** — build pipeline, ember-data → WarpDrive, Glint adoption. |

**Ember 3.x (Feb 2018 – 3.28 LTS Aug 2021) — mixed classic/Octane era**

| Skill | Purpose |
|---|---|
| `ember-3-mixed-classic-octane` | Reading and maintaining apps where some files are `Ember.Component.extend(...)` and others are `class extends Component`. |
| `ember-3-octane-adoption` | Adopting Octane *within* a 3.16+ app — opt-in features, codemods, per-file conversion order. |
| `ember-3-recommendations` | Getting the most out of 3.28 LTS as a checkpoint, deprecation hygiene, addon picks. |
| `ember-3-to-4-migration` | The path **3.28 → 4.12 LTS** — what gets removed, jQuery final removal, ember-data 4 typing. |

**Ember 2.x (Aug 2015 – 2.18 LTS Dec 2017) — classic-only era**

| Skill | Purpose |
|---|---|
| `ember-2-classic-patterns` | `Ember.Object.extend`, `Ember.computed`, `Ember.Component`, `actions: {...}`, mixins, observers, two-way bindings. |
| `ember-2-testing` | The pre-Octane test API: `moduleFor`, `moduleForComponent`, `moduleForAcceptance`, the bridge to `setupTest`. |
| `ember-2-recommendations` | If you must stay on 2.x — pinning, security, dependency hygiene, what to backport. |
| `ember-2-to-3-migration` | The path **2.18 LTS → 3.28 LTS**, in order, with `ember-cli-update` and the addon survival list. |

### Agents

| Agent | Use for |
|---|---|
| `ember-architect` | Designing Ember apps — feature layout, routing/state shape, addon selection, migration plans. |
| `ember-test-engineer` | Test strategy and authoring — QUnit, page objects, Mirage scenarios, a11y, performance assertions. |
| `ember-2-migrator` | Drives a 2.18 → 3.28 upgrade end-to-end: deprecation triage, addon replacement, codemod scheduling. |
| `ember-3-migrator` | Drives Octane adoption within 3.x and the 3.28 → 4.12 LTS jump. |
| `ember-4-migrator` | Drives the 4.12 → 5.x jump: Embroider settling, ember-data typing tightening, Glint introduction. |

### Commands

| Command | Action |
|---|---|
| `/ember-component` | Scaffold a Glimmer component with template, backing class, and integration test. |
| `/ember-route` | Scaffold a route with `model` hook, controller (if needed), template, and application test. |
| `/ember-service` | Scaffold a service with TS types and a unit test. |
| `/ember-test` | Run the test suite (QUnit) and triage failures. |

## Version coverage

| `ember-source` in `package.json` | Era | Primary skills |
|---|---|---|
| `5.x` or later | Octane + Polaris (current) | Modern skills above. |
| `4.x` | Octane-only | `ember-4-*` skills + modern fundamentals. |
| `3.x` | Mixed classic/Octane | `ember-3-*` skills + Octane fundamentals (3.15+). |
| `2.x` | Classic-only | `ember-2-*` skills. |

## End-to-end migration path

```
Ember 2.18 LTS ──→ Ember 3.28 LTS ──→ Ember 4.12 LTS ──→ Ember 5.x latest
       │                  │                  │                  │
       ↓                  ↓                  ↓                  ↓
  2-migrator         3-migrator         4-migrator         architect
  2 → 3 skill        3 → 4 skill        4 → 5 skill        polaris skill
```

For a 2.x → modern upgrade:

1. Run the `ember-2-migrator` agent. Land at 3.28 LTS.
2. Run the `ember-3-migrator` agent (Octane adoption, then 3 → 4). Land at 4.12 LTS.
3. Run the `ember-4-migrator` agent. Land at the latest 5.x.
4. Use `ember-architect` and the modern skills from there.

Each migrator agent refuses to start until the previous phase's prerequisites are verifiable, so it's hard to skip a step by accident.

You **never** jump multiple major versions in one commit. Walk through the LTS releases, fixing deprecations as you go.

## Should I stay on an older version?

| Situation | Recommendation |
|---|---|
| Greenfield app | No — use the latest LTS. |
| Active product, paying customers, on `<5.x` | Plan an upgrade. Releases after your version have shipped years of bug fixes and security patches. |
| 2.18 internal tool, low traffic, no auth/PCI surface | You can hold, but pin every dep, run `npm audit` on a schedule, budget a quarter for the eventual jump. |
| 3.28 LTS healthy team | Plan the 4.x jump. 3.28 stopped getting non-security patches in 2022, security patches in 2023. |
| 4.12 LTS healthy team | Plan a 5.x bump. 4.12 LTS support window has ended for non-security patches. |
| Regulated/air-gapped, change-is-expensive | Stay, but isolate. Treat the app as a black box; don't extend features beyond bug fixes. |

There is **no security backporting** to Ember 2.x. Vulnerabilities in Ember itself, its `jquery` peer dep, and the addons of that era are your problem to detect and patch.

## Layout

```
ember-claude-skills/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── CLAUDE.md                       # authoring contract for this plugin
├── README.md
├── agents/
│   ├── ember-architect.md
│   ├── ember-test-engineer.md
│   └── ember-{2,3,4}-migrator.md
├── commands/
│   └── ember-{component,route,service,test}.md
└── skills/
    └── ember-*/
        ├── SKILL.md                # thin index + core constraints
        └── references/*.md         # supporting files (where applicable)
```

Skills follow the **progressive-disclosure** pattern: `SKILL.md` is loaded when its description matches, and `references/*.md` are pulled in only when the agent navigates there. Today only `ember-ecosystem-addons` has supporting files — its 16 addon entries are split into category references (`auth.md`, `async.md`, `testing.md`, `forms-and-ui.md`, `i18n.md`, `observability.md`, `build-and-lint.md`).

## Sources

This plugin draws from:

- The official [Ember Guides](https://guides.emberjs.com) and [API docs](https://api.emberjs.com).
- The [Ember RFCs](https://rfcs.emberjs.com) — especially Octane (RFC #176) and Polaris-track RFCs (#779 template tag, #496 `<template>` tag, #756 strict mode).
- Reference repositories: [ember-source](https://github.com/emberjs/ember.js), [ember-data](https://github.com/emberjs/data), [ember-cli](https://github.com/ember-cli/ember-cli), [glimmer-vm](https://github.com/glimmerjs/glimmer-vm), and the [empress/field-guide](https://github.com/empress) examples.
- Community blueprints from [emberobserver.com](https://emberobserver.com) — addons listed in `skills/ember-ecosystem-addons/SKILL.md`.
- Official deprecation guidance at [deprecations.emberjs.com](https://deprecations.emberjs.com) for each version migration.
