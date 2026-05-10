---
description: Detect the project's ember-source version and dispatch to the right migrator agent (ember-2-migrator / ember-3-migrator / ember-4-migrator). Refuses to skip LTS hops.
---

# /ember-upgrade

Single entry point for "upgrade my Ember app." Reads `ember-source` from `package.json`, picks the correct migrator agent, and hands off with the right starting context.

## Usage

```
/ember-upgrade [--target=<version>] [--dry-run]
```

- `--target=<version>`: stop at this version instead of the latest LTS. Useful for a "hop one LTS at a time" workflow.
- `--dry-run`: print the dispatch plan and the agent invocation but do not actually start the migration.

## Steps

1. **Locate `package.json`.** From the working directory, walk up if needed.
2. **Read the current Ember version.** Look at `devDependencies['ember-source']` (most projects) or `dependencies['ember-source']` (rare). Strip range qualifiers (`^`, `~`).
3. **Read the lockfile** (`pnpm-lock.yaml` / `yarn.lock` / `package-lock.json`) to confirm the resolved version. The range in `package.json` is not authoritative.
4. **Detect a worktree.** Run `git rev-parse --show-toplevel` and `git worktree list`. If the user is on `main` (or the only worktree), warn that migrations rewrite many files and recommend `claude --worktree`.
5. **Pick the migrator** using the dispatch table below.
6. **Hand off.** Print: detected version, target version, chosen agent, the first thing the agent will do. If `--dry-run`, stop here. Otherwise invoke the agent (`Use the ember-N-migrator agent to ...`).

## Dispatch table

| Detected `ember-source` | Agent to invoke | First hop |
|---|---|---|
| `2.0.x` – `2.18.x` | `ember-2-migrator` | 2.18 LTS → 3.4 LTS, then walk the 3.x chain to 3.28. |
| `3.0.x` – `3.15.x` | `ember-3-migrator` | Reach 3.16 first, then begin Octane adoption. |
| `3.16.x` – `3.28.x` | `ember-3-migrator` | Finish Octane adoption on 3.28, then 3.28 → 4.4 → 4.8 → 4.12. |
| `4.0.x` – `4.11.x` | `ember-4-migrator` | Reach 4.12 LTS first. |
| `4.12.x` | `ember-4-migrator` | Confirm Embroider + TS + addon hygiene, then 4.12 → 5.x LTS. |
| `5.0.x` – latest | (no migrator) | Recommend `ember-architect` agent + the `ember-polaris-migration` skill. |

If the version is between major eras (e.g. `3.28` → `4.0` mid-flight, with `ember-source` already bumped but deprecations unaddressed), prefer the *higher* migrator — it is responsible for confirming the previous era's exit criteria.

## Refusal cases

- **Skipping LTS hops.** If the user passes `--target=5.4` and the current version is `2.18`, refuse and show the LTS chain. Walking each LTS catches deprecations the next hop expects to be gone.
- **Dirty working tree.** If `git status --porcelain` is non-empty, refuse and ask the user to commit or stash. Migrators will rewrite many files and need a clean baseline for diffs.
- **Unsupported version.** If the version is below 2.0 or unrecognized, do not dispatch — print the detected version and ask the user how to proceed.

## Output format

```
/ember-upgrade — detected ember-source 3.24.7 (lockfile: 3.24.7)

  Era:           Ember 3.x (mixed classic/Octane)
  Target:        4.12 LTS (latest 4.x)
  Path:          3.24 → 3.28 LTS → 4.4 → 4.8 → 4.12
  Migrator:      ember-3-migrator
  Worktree:      ⚠ none — recommend `claude --worktree` before proceeding

  First step:    bump 3.24 → 3.28 LTS via ember-cli-update, run codemods,
                 fix deprecations until the suite is green.

Dispatching ember-3-migrator …
```

For `--dry-run`, stop after `Dispatching ember-3-migrator …` is replaced with `(dry-run — no agent started).`

## Conventions enforced

- One migrator per major-version range. Do not call multiple migrators in one session — each refuses to start until the previous phase's exit criteria are verifiable.
- Always read the lockfile, never trust the range in `package.json` alone.
- Always recommend a worktree for migrations; never *force* one (the user may be in a CI sandbox or already in a branch).
- Never propose `--no-verify` to bypass a failing migrator hook. If the migrator stops, the failure is the next thing to fix.

## See also

- Agent: `ember-2-migrator` — 2.18 → 3.28.
- Agent: `ember-3-migrator` — Octane adoption + 3.28 → 4.12.
- Agent: `ember-4-migrator` — 4.12 → 5.x.
- Skill: `ember-2-to-3-migration`, `ember-3-to-4-migration`, `ember-4-to-5-migration`.
- Skill: `ember-polaris-migration` (post-5.x).
