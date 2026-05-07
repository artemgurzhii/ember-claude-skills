---
name: ember-2-recommendations
description: Practical advice for teams that must stay on Ember 2.x for now — pinning, security hygiene, what to backport, what to leave alone, when to declare upgrade-impossible-without-rewrite. Use when triaging a frozen 2.x codebase, when justifying upgrade budget, or when scoping the smallest viable maintenance footprint.
type: feedback
---

# Ember 2.x — Recommendations for Teams Who Must Stay

If you genuinely cannot upgrade right now, follow these rules to limit the blast radius. This is **not** a substitute for upgrading; it's a holding pattern.

## Be honest about why you're stuck

Most "we can't upgrade" stories fall into one of three buckets. The right action depends on which:

| Reason | Real path forward |
|---|---|
| "We've never budgeted for it." | Plan a migration. The longer you wait, the more it costs. Use `ember-3-legacy/skills/ember-2-to-3-migration` to scope. |
| "We have ~50 abandoned addons and one of them does X." | Identify the actual blocker — usually 1–3 addons. Fork them and modernize. |
| "Air-gapped/regulated/customer-bound." | Stay, but treat the codebase as a frozen artifact. No new features. Keep a small lights-on team. |

If your reason is bucket 1 or 2, **escalate to a real upgrade**. Don't try to make a 2.x app a long-term home.

## Pin everything

`package.json` should have **exact versions** (`^` and `~` removed) for:

- `ember-source`
- `ember-cli`
- `ember-data`
- `ember-cli-htmlbars`
- All `ember-*` addons
- `jquery`
- `qunit` and `ember-qunit`
- Build-time deps: `broccoli-*`, `loader.js`, `ember-cli-babel`

Commit a lockfile (`package-lock.json` or `yarn.lock`) and don't `npm update` it. A `yarn install --frozen-lockfile` (or `npm ci`) on a clean machine should reproduce your build exactly.

## Security hygiene

Ember 2.x itself **is no longer receiving security patches**. The risk surface is broader than just Ember:

| Surface | What to do |
|---|---|
| Ember (core) | Read [emberjs/ember.js security advisories](https://github.com/emberjs/ember.js/security/advisories) for CVEs that apply to your minor. Backport patches yourself if you must — and document the fork. |
| `jquery` | Pin to a fixed version, *but* watch CVE feeds. jQuery 2.x has known XSS issues; jQuery 3 is safer but may need code changes. |
| Ember Data | Same as Ember core. |
| Addons | Most 2.x-era addons are abandoned. Run `npm audit` and prefer transitive-pin overrides for vulnerable sub-deps. |
| Node version | Whatever Node version `ember-cli` of your era requires. Pin in `engines` and in CI. |

Set up [Dependabot](https://github.com/dependabot) or Renovate scoped to security-only updates. Don't auto-merge — every patch in a 2.x app needs a human deciding whether it actually applies.

## What to backport, what to leave alone

| Type of fix | Backport? |
|---|---|
| Reflected/stored XSS in your own code | Yes. Always. |
| Dependency CVE with a known exploit | Yes, ideally via transitive override. |
| Dependency CVE with no known exploit, low CVSS | Triage. Document the decision. |
| Performance optimization | No. Risk of regression > benefit. |
| Refactor/cleanup | No. Holding pattern means *no* drive-by changes. |
| New feature | No. Scope only bug fixes and security. |

Every commit to a frozen 2.x app should have a clear "why now" justification.

## Test coverage is your seat belt

If your test suite is thin, your only safe move is to *add* tests, not change code. Aim for:

- Application tests for the top 5 user flows.
- Mirage scenarios for every endpoint the app calls.
- `data-test-*` selectors via `ember-test-selectors` — this also pre-pays the migration cost.

Without tests, even a security backport is dangerous.

## Ember Inspector still works

Browser extension is version-aware and works back to early 2.x. Use it for "what state is this controller actually in" debugging. https://github.com/emberjs/ember-inspector/releases — you may need an older release of the extension if your Ember is very old.

## Things that age badly in 2.x apps

Watch for these — they are early warning signs of an "upgrade now or rewrite" decision:

- **Browser support drifting away.** Ember 2.x's polyfills target browsers that no longer exist. Your prod traffic moved on.
- **Build times exceeding 60s on a fast machine.** Broccoli/Babel of that era doesn't tree-shake; CPUs got faster, your build didn't.
- **`broccoli-asset-rev` and friends fail with newer Node.** Pin Node aggressively.
- **`fingerprint` plugins that break on `Node 20+`.** This forces you onto an unsupported Node, which has its own CVE risk.
- **An addon's GitHub repo is archived.** Fork it now, before the URL stops resolving.

## When to declare "rewrite, not upgrade"

If multiple of these are true, an upgrade through the LTS chain is likely **more expensive** than a rewrite into a current Ember:

- < 30% test coverage and no test culture.
- Heavy reliance on archived/forked addons.
- Custom resolver or build hacks.
- App startup time > 5s on a modern laptop.
- Active product roadmap that keeps adding to the 2.x app.

In that case, scope a rewrite into a current Ember LTS using the [`ember/`](../../../ember) plugin, run both apps in parallel behind a path-based proxy, and migrate routes one at a time.

## What you can keep doing

Some patterns from 2.x are still fine in modern Ember and don't need to change just because you upgrade:

- Routing structure (`Router.map`, route hooks).
- Mirage scenarios.
- Service-style singletons (just rename `extend` later).
- `ember-power-select`, `ember-cli-mirage`, `ember-test-selectors`, `ember-cli-page-object` — these have evolved alongside Ember.
- Application architecture (route → component decomposition).

The migration is largely **mechanical** — the architecture rarely needs rethinking, just the syntax.

## Verification — am I in a safe holding pattern?

- [ ] All Ember-related deps are pinned to exact versions.
- [ ] The lockfile is committed.
- [ ] CI runs `npm audit --omit=dev` (or equivalent) and you've triaged every finding.
- [ ] You have a documented Node version with a CVE-clean release.
- [ ] You can produce a clean build from a fresh checkout, today, with no `node-gyp` errors.
- [ ] Top 5 user flows are covered by application tests.
- [ ] You have a written upgrade ETA, even if it's "Q4 next year."
- [ ] Nobody is adding new features to the 2.x app without a new-features-elsewhere plan.

If any of these are unchecked, you're not in a holding pattern — you're in a slow-motion incident.

## See also

- `ember-2-to-3-migration` — when you're ready to start.
- `ember-2-classic-patterns` / `ember-2-testing` — what to read while triaging.
- [`ember/`](../../../ember) — the destination.
