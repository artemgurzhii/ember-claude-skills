---
description: Run the Ember test suite (QUnit), triage any failures, and (when asked) author additional tests for missing coverage.
---

# /ember-test

Run the Ember test suite, summarize results, and triage failures.

## Usage

```
/ember-test [--filter=<pattern>] [--module=<name>] [--once]
```

- `--filter=<pattern>`: pass through to QUnit's filter (substring match on test names).
- `--module=<name>`: only run tests whose module starts with this name.
- `--once`: single run (no watch). Default.

## Steps

1. **Detect the runner.** Look at `package.json` for `scripts.test`. Common patterns:
   - `pnpm test` / `npm test` (often runs `ember test`).
   - `ember test --once`.
   - `ember exam` (parallelized).
2. **Run the suite** with the requested filters.
3. **If everything passes:** report green and the count.
4. **If there are failures:**
   - For each failing test, capture the assertion message and stack trace.
   - Group failures by likely cause:
     - **Settled-state issues** ("a test ended before all promises settled") — point at unawaited work; suggest `await waitForPromise` from `@ember/test-waiters` or moving work into an `ember-concurrency` task.
     - **DOM not rendering as expected** — diff the assertion against the current template; check whether the relevant `@tracked` field is being mutated correctly.
     - **Mirage 404 / missing handler** — list the requested URL; suggest the missing route handler.
     - **Auth failures** — check whether the test set up `session.authenticate` or stubbed the session service.
5. **Propose fixes.** For each failure, give a one-paragraph diagnosis and the smallest patch.

## When asked to write tests for missing coverage

1. **Read the code under test** (component / route / service / model).
2. **Decide the test type** using the matrix from the `ember-test-engineer` agent:
   - Service → unit test.
   - Component → rendering test.
   - Route or multi-step flow → application test.
3. **Add `data-test-*` attributes** to the relevant template before writing assertions.
4. **Write the smallest test that fails without the implementation** (or the smallest set proving the behavior).
5. **Run the test, watch it fail meaningfully, then re-run to confirm green.**

## Conventions enforced

- No `setTimeout` / `Promise.resolve().then(...)` in tests — only `@ember/test-helpers` async helpers.
- Selectors are `[data-test-*]`, never CSS classes.
- DOM assertions use `assert.dom(...)` (qunit-dom).
- Mirage is used for any test that hits the network.
- One concept per test, named as a sentence.

## Output format

Always end the run with a one-line summary:

```
PASS  124 tests  in 8.4s
```

or

```
FAIL  3/124 tests
  - Integration | Component | post-card > shows the publish date  (DOM mismatch)
  - Acceptance | dashboard > redirects when unauthenticated  (router state)
  - Unit | Service | cart > subtotal sums item prices  (assertion)
```

For each failure, follow with the diagnosis and the proposed patch (or the file paths to inspect).

## See also

- Skill: `ember-testing`
- Agent: `ember-test-engineer`
- Skill: `ember-ecosystem-addons` → `ember-cli-mirage`, `ember-test-selectors`, `ember-cli-page-object`
