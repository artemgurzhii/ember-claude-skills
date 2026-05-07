# ember-4-legacy

A Claude Code plugin for working in **Ember 4.x** codebases (released January 2022, final release **4.12 LTS** August 2023).

4.x is the **Octane-only** era. The classic API was finally removed. jQuery was finally removed. `Ember.X` globals were finally gone. The mental model is exactly the modern one — but the build pipeline, TypeScript story, and template authoring (`<template>` tag) all matured between 4.x and 5.x.

## Should you stay on 4.x?

| Situation | Recommendation |
|---|---|
| 4.12 LTS, healthy team | Plan a 5.x bump. 4.12 LTS support window has ended for non-security patches; security backports are rare. |
| 4.4 / 4.8 LTS | Get to 4.12 first, then jump to 5.x. |
| TypeScript adopted via `ember-cli-typescript` | Plan to migrate to the official Ember TS support, which arrived in 4.x without that addon, and to Glint in 5.x. |
| Embroider not yet adopted | If you're staying on 4.x, this is the moment. Compat mode → optimized as confidence builds. |
| New app in 2026 | No — go straight to the latest LTS via the [`ember/`](../ember) plugin. |

## What's inside

### Skills
| Skill | Purpose |
|---|---|
| `ember-4-octane-only` | What 4.x removed and what you can rely on — no classic anything, jQuery gone, Ember.X gone, observers gone. |
| `ember-4-typescript-early` | Adopting TypeScript on 4.x without Glint (early-Glint is fragile here), the Registry pattern, where `ember-cli-typescript` fits, and what to defer to 5.x. |
| `ember-4-recommendations` | Picking 4.12 as a checkpoint, Embroider adoption, addon hygiene, when to press on. |
| `ember-4-to-5-migration` | The path **4.12 → 5.x latest**, with the build-pipeline assumptions, ember-data → WarpDrive transition, and Glint adoption. |

### Agent
| Agent | Use for |
|---|---|
| `ember-4-migrator` | Drives the 4.12 → 5.x jump, including Embroider settling, ember-data typing tightening, and (optionally) Glint introduction. |

## Migration path overview

```
(from ember-3-legacy)  →  4.4 LTS  →  4.8 LTS  →  4.12 LTS
                                                      ↓
                                                    5.4  →  5.8  →  5.12  →  ...latest
                                                                                ↓
                                                                              (continue with `ember/` plugin)
```

5.x doesn't follow the same predictable LTS cadence as earlier versions in some respects, but the LTS-by-LTS pattern is still the safest path. The `ember-4-migrator` agent walks it.

## See also

- [`ember-3-legacy/`](../ember-3-legacy) — the previous plugin, for getting *to* 4.x.
- [`ember/`](../ember) — the destination edition (Octane + Polaris).
