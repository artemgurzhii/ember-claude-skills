# ember-3-legacy

A Claude Code plugin for working in **Ember 3.x** codebases (released February 2018, final release **3.28 LTS** August 2021).

3.x is the era where modern Ember took shape:

- **3.4 LTS** — angle-bracket invocation (`<UserCard>`).
- **3.10** — native classes officially supported.
- **3.13** — `@tracked` and `@glimmer/component` available behind a feature flag.
- **3.15** — **Octane edition** becomes default. Native classes, decorators, modifiers, no two-way bindings.
- **3.20** — jQuery integration disabled by default for new apps.
- **3.28 LTS** — the last 3.x. Where most "we're upgrading" projects hit a checkpoint.

## Should you stay on 3.x?

| Situation | Recommendation |
|---|---|
| Active 3.28 LTS app, healthy team | Plan the 4.x jump. 3.28 stopped getting non-security patches in 2022, security patches in 2023. |
| Active 3.x app < 3.28 | First get to 3.28 (use the [`ember-2-legacy/skills/ember-2-to-3-migration`](../ember-2-legacy/skills/ember-2-to-3-migration) skill or just the deprecation loop). Then plan 3 → 4. |
| Mid-Octane adoption on 3.16-3.28 | Keep going — Octane within 3.x is the same Octane as 4.x and 5.x. |
| New app on 3.x in 2026 | No. |

## What's inside

### Skills
| Skill | Purpose |
|---|---|
| `ember-3-mixed-classic-octane` | Reading and maintaining apps where some files are `Ember.Component.extend(...)` and others are `class extends Component`. The half-migrated reality. |
| `ember-3-octane-adoption` | Adopting Octane *within* a 3.16+ app — opt-in features, codemods, the per-file conversion order. |
| `ember-3-recommendations` | Getting the most out of 3.28 LTS as a checkpoint, deprecation hygiene, addon picks. |
| `ember-3-to-4-migration` | The path **3.28 → 4.12 LTS** — what gets removed, jQuery final removal, ember-data 4 typing, addon survival. |

### Agent
| Agent | Use for |
|---|---|
| `ember-3-migrator` | Drives Octane adoption within 3.x and the 3.28 → 4.12 LTS jump. |

## Migration path overview

```
(from ember-2-legacy)  →  3.4 LTS  →  ...  →  3.28 LTS  ─┐
                                                          │ (Octane adoption happens during 3.16-3.28)
                                                          ↓
                                                       4.4 LTS  →  4.8 LTS  →  4.12 LTS
                                                                                  ↓
                                                                  (continue with ember-4-legacy plugin)
```

## See also

- [`ember-2-legacy/`](../ember-2-legacy) — the previous plugin, for getting *to* 3.x.
- [`ember-4-legacy/`](../ember-4-legacy) — the next plugin, for getting *out of* 3.x.
- [`ember/`](../ember) — the destination edition (Octane + Polaris).
