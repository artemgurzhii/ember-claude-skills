# EmberJS claude skills

Personal collection of Claude Code skills and plugins maintained by [@artemhurzhii](https://github.com/artemhurzhii).

Each top-level folder is a self-contained plugin following the Claude Code plugin layout (`.claude-plugin/plugin.json` + `skills/`, `agents/`, optionally `commands/`).

## Plugins

### Ember.js (full version-history coverage)

| Plugin | Era | What it covers |
|---|---|---|
| [`ember/`](./ember) | **Octane + Polaris (current)** | Modern Ember authoring — Octane fundamentals, components, routing, services, Ember Data, testing, ecosystem addons, TypeScript/Glint, the Polaris transition (`<template>` tag). **Use for any new Ember code.** |
| [`ember-4-legacy/`](./ember-4-legacy) | Ember 4.x (Jan 2022 – 4.12 LTS Aug 2023) | Octane-only era — jQuery removed, classic API gone, official TS, Embroider matures. Includes 4 → 5 migration. |
| [`ember-3-legacy/`](./ember-3-legacy) | Ember 3.x (Feb 2018 – 3.28 LTS Aug 2021) | The mixed classic/Octane era. Octane became default at 3.15. Includes Octane adoption within 3.x and 3 → 4 migration. |
| [`ember-2-legacy/`](./ember-2-legacy) | Ember 2.x (Aug 2015 – 2.18 LTS Dec 2017) | Classic-only era — `Ember.Object.extend`, mixins, two-way bindings, jQuery, classic test API. Includes 2 → 3 migration. |

### Which Ember plugin should I install?

Look at the app's `package.json` `ember-source` version and pick the matching plugin:

| `ember-source` | Plugin |
|---|---|
| `5.x` or later | [`ember/`](./ember) |
| `4.x` | [`ember-4-legacy/`](./ember-4-legacy) |
| `3.x` | [`ember-3-legacy/`](./ember-3-legacy) |
| `2.x` | [`ember-2-legacy/`](./ember-2-legacy) |
| New project | [`ember/`](./ember) |

You can install **multiple plugins at once** — the legacy plugins don't conflict with the modern one. A typical mid-migration setup might have both [`ember-3-legacy/`](./ember-3-legacy) (for the codebase as it stands today) and [`ember/`](./ember) (as the destination reference) loaded side-by-side.

### End-to-end migration path

Each legacy plugin includes a migrator agent and a "→ next" migration skill that hands off to the next plugin. The full chain:

```
ember-2-legacy ──→ ember-3-legacy ──→ ember-4-legacy ──→ ember
   (2.18 LTS)        (3.28 LTS)         (4.12 LTS)      (5.x latest)
        │                  │                  │             │
        ↓                  ↓                  ↓             ↓
   2-migrator        3-migrator         4-migrator      architect
   2 → 3 skill       3 → 4 skill        4 → 5 skill     polaris skill
```

For a 2.x → modern migration:

1. Install [`ember-2-legacy/`](./ember-2-legacy). Run the `ember-2-migrator` agent. Land at 3.28 LTS.
2. Install [`ember-3-legacy/`](./ember-3-legacy). Run the `ember-3-migrator` agent (Octane phase, then 3 → 4 hops). Land at 4.12 LTS.
3. Install [`ember-4-legacy/`](./ember-4-legacy). Run the `ember-4-migrator` agent. Land at the latest 5.x LTS.
4. Install [`ember/`](./ember). Use `ember-architect` and the modern skills for everything from there.

Each agent refuses to start until the previous phase's prerequisites are verifiable, so it's hard to skip a step by accident.

## Installing a plugin locally

From a Claude Code session:

```
/plugin install /Users/artemhurzhii/Programming/Work/ember-claude-skills/<plugin-folder>
```

You can also point Claude Code at the directory directly via a marketplace manifest.

## Layout convention

```
<plugin>/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # (optional) marketplace metadata
├── README.md
├── skills/<skill-name>/SKILL.md
├── agents/<agent-name>.md
└── commands/<command-name>.md   (optional)
```
