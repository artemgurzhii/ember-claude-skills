# ember-2-legacy

A Claude Code plugin for working in **Ember 2.x** codebases (released August 2015, final release **2.18 LTS** December 2017).

If you're starting a new Ember app in 2026, you do **not** want this plugin — install [`ember/`](../ember) instead. This plugin exists for two audiences:

1. Maintainers of frozen 2.x apps that can't be upgraded right now (regulated industry, internal tool with no budget for modernization).
2. Anyone running a **2 → 3 → 4 → 5 → 6** upgrade and needing to read/understand the starting state.

## Should you stay on 2.x?

| Situation | Recommendation |
|---|---|
| Greenfield app | Don't. Use the latest LTS. |
| Active product, paying customers | Plan an upgrade. Every release after 2.18 has shipped 5+ years of bug fixes and security patches. |
| Internal tool, low traffic, no auth/PCI surface | You can hold, but pin every dep, run `npm audit` on a schedule, and budget a quarter for the eventual jump. |
| Regulated/air-gapped, change-is-expensive | Stay, but isolate. Treat the app as a black box; don't extend features beyond bug fixes. |

There is **no security backporting** to Ember 2.x. Vulnerabilities in Ember itself, its `jquery` peer dep, and the addons of that era are your problem to detect and patch.

## What's inside

### Skills
| Skill | Purpose |
|---|---|
| `ember-2-classic-patterns` | `Ember.Object.extend`, `Ember.computed`, `Ember.Component`, `actions: {...}`, mixins, observers, two-way bindings — the canonical 2.x mental model. |
| `ember-2-testing` | The pre-Octane test API: `moduleFor`, `moduleForComponent`, `moduleForAcceptance`, `ember-cli-qunit`, the bridge to `setupTest`. |
| `ember-2-recommendations` | If you must stay on 2.x — pinning, security, dependency hygiene, what to backport, what to leave alone. |
| `ember-2-to-3-migration` | The path **2.18 LTS → 3.28 LTS**, in order, with the deprecation workflow, `ember-cli-update`, and the addon survival list. |

### Agent
| Agent | Use for |
|---|---|
| `ember-2-migrator` | Drives a 2.18 → 3.28 upgrade end-to-end: deprecation triage, addon replacement, codemod scheduling, regression catch. |

## Migration path overview (one-liner)

```
2.18 LTS  →  3.4 LTS  →  3.8 LTS  →  3.12 LTS  →  3.16 LTS  →  3.20 LTS  →  3.24 LTS  →  3.28 LTS
                                                                                            ↓
                                                                      (continue with ember-3-legacy plugin)
```

You **never** jump 2 → 3.28 in one commit. Walk through the LTS releases, fixing deprecations as you go.

## See also

- [`ember/`](../ember) — the destination edition (Octane + Polaris).
- [`ember-3-legacy/`](../ember-3-legacy) — the next plugin to use after you finish the 2 → 3 jump.
- Official: [Ember 2.18 LTS release post](https://blog.emberjs.com/), [Deprecation guide](https://deprecations.emberjs.com).
