---
name: ember-2-testing
description: The Ember 2.x test API — moduleFor, moduleForComponent, moduleForModel, moduleForAcceptance, the global async helpers (visit, click, fillIn), wait(), and the bridge to the modern setupTest/setupRenderingTest API introduced in 3.x. Use when reading or fixing 2.x tests, or when planning the conversion to the modern qunit-based API.
type: reference
---

# Ember 2.x Testing

The 2.x test API was overhauled in 3.x. If you're reading a test file and you see `moduleFor`, `moduleForComponent`, `andThen`, or global async helpers without `await`, you're in the 2.x dialect.

## Test types and their setup helpers

| What you're testing | 2.x helper | 3.x+ replacement |
|---|---|---|
| A unit (service, util, controller class) | `moduleFor('service:foo', { ... })` | `setupTest(hooks)` (then `this.owner.lookup('service:foo')`) |
| A component (in isolation, with a render context) | `moduleForComponent('user-card', { integration: true })` | `setupRenderingTest(hooks)` |
| A model (Ember Data) | `moduleForModel('post', ...)` | `setupTest(hooks)` |
| A user flow (full app boot) | `moduleForAcceptance('/posts')` | `setupApplicationTest(hooks)` |

These helpers come from `ember-qunit` (and `ember-cli-qunit`/`ember-cli-mocha` in some 2.x setups).

## Anatomy of a 2.x acceptance test

```js
// tests/acceptance/posts-test.js
import { test } from 'qunit';
import moduleForAcceptance from 'my-app/tests/helpers/module-for-acceptance';

moduleForAcceptance('Acceptance | posts');

test('visiting /posts', function (assert) {
  visit('/posts');

  andThen(function () {
    assert.equal(currentURL(), '/posts');
    assert.equal(find('.post').length, 3);
  });
});
```

Key 2.x flavors:

- **Global async helpers**: `visit`, `click`, `fillIn`, `keyEvent`, `triggerEvent`, `find`, `currentURL`, `currentRouteName`. They are *globally injected*, not imported.
- **`andThen(fn)`**: waits for the test to settle, then runs `fn`. Each `andThen` block can issue more async helpers, which are themselves followed by another `andThen`. This is the 2.x equivalent of `await`.
- **`wait()`**: returns a promise that resolves when settled. Useful when you need a promise outside `andThen`.

The post-2.x style — `await visit('/')` — is the same idea, but with native `async/await` and imports from `@ember/test-helpers`.

## A 2.x component (integration) test

```js
import { moduleForComponent, test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

moduleForComponent('user-card', 'Integration | Component | user-card', {
  integration: true
});

test('renders the user name', function (assert) {
  this.set('user', { name: 'Ada Lovelace' });
  this.render(hbs`{{user-card user=user}}`);
  assert.equal(this.$('.user-card .name').text().trim(), 'Ada Lovelace');
});
```

Notes:

- **`integration: true`** flips the helper into rendering mode. Without it, you get a unit-style stub container with no render context.
- **`this.render(hbs\`...\`)`** is the 2.x render call. In 3.x+, it's `await render(hbs\`...\`)` from `@ember/test-helpers`.
- **`this.$(...)`** is jQuery. `qunit-dom` (`assert.dom(...)`) didn't exist yet — assertions are usually `assert.equal(this.$('.x').text().trim(), 'expected')`.
- Curly invocation in the template (`{{user-card user=user}}`). Angle-bracket invocation worked from 3.4 onward.

## A 2.x unit test

```js
import { moduleFor, test } from 'ember-qunit';

moduleFor('service:cart', 'Unit | Service | cart', {
  // dependencies you need resolved:
  needs: ['service:notifications']
});

test('adds an item', function (assert) {
  const service = this.subject();
  service.add({ id: 1, price: 10 });
  assert.equal(service.get('items.length'), 1);
});
```

- **`needs: [...]`**: declares which dependencies the resolver should make available. In 3.x+, this dance is replaced by `this.owner.register(...)` for fakes and automatic resolution of the rest.
- **`this.subject()`**: instantiates the thing under test. In 3.x+, you write `this.owner.lookup('service:cart')`.

## `wait()` and pending work

`wait()` from `ember-test-helpers` resolves when:

- All promises tracked by the test container are resolved.
- No pending AJAX requests (via `jQuery.active`).
- No timers in the run loop.

If a test hangs forever in `andThen`, almost always it's because some async work isn't registered — usually a raw `setTimeout`, a `fetch` (jQuery's AJAX is tracked, `fetch` isn't), or a promise created outside the run loop. **Test waiters** (`Ember.Test.registerWaiter(fn)`) let you teach the framework to wait for arbitrary work; modern tests use `@ember/test-waiters` instead.

## The bridge — running both APIs side-by-side

Most apps modernize one test file at a time. To make this safe:

- In 2.x apps near the end of life, install `ember-qunit@^4` (which supports both `moduleFor*` and `setupTest`).
- Convert one file at a time: replace `moduleForXxx` with the matching `module(...) { setupXxx(hooks); }` block.
- Replace each `andThen(...)` with `await` on the preceding helper.
- Replace `this.$(...)` with `find(...)` or `assert.dom(...)`.

Example before/after:

```js
// 2.x style
moduleForAcceptance('Acceptance | posts');

test('visiting /posts', function (assert) {
  visit('/posts');
  andThen(() => assert.equal(currentURL(), '/posts'));
});
```

```js
// modernized
import { module, test } from 'qunit';
import { setupApplicationTest } from 'ember-qunit';
import { visit, currentURL } from '@ember/test-helpers';

module('Acceptance | posts', function (hooks) {
  setupApplicationTest(hooks);

  test('visiting /posts', async function (assert) {
    await visit('/posts');
    assert.strictEqual(currentURL(), '/posts');
  });
});
```

Both can coexist in the same suite — convert at your own pace.

## Selectors

`ember-test-selectors` (the `data-test-*` strip-on-build addon) was already common in 2.x. If your codebase doesn't use it, *adding* `data-test-*` selectors during the migration is one of the higher-leverage things you can do — your tests become independent of the CSS overhaul that almost always accompanies a major upgrade.

## Mirage — already a thing

`ember-cli-mirage` was the standard mock-server addon throughout 2.x. The factory/route-handler API has been remarkably stable; in most cases, your 2.x Mirage setup will keep working after the upgrade with only minor tweaks.

## Common 2.x test mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Test passes locally, fails in CI | Order-dependent state (a previous test's controller bled in). | Tear down state in `afterEach`; convert to `module + hooks`. |
| `andThen` block runs before async work finishes | `wait()` doesn't see the work because it's outside the run loop. | Wrap in `Ember.run(...)` or register a test waiter. |
| jQuery DOM lookups break after CSS refactor | Selectors are class-based. | Add `data-test-*` and select by those. |
| `this.subject()` returns `undefined` | Missing `needs:` declaration. | Add the dependency factory name. |
| Acceptance test hangs forever | Unawaited `fetch` or `setTimeout` not registered with a waiter. | Register a waiter via `Ember.Test.registerWaiter(...)`. |

## Verification

- [ ] You recognize `moduleFor*` as the 2.x setup mechanism.
- [ ] You can read `andThen`-style tests as the pre-`await` shape.
- [ ] You know `this.$(...)` is jQuery and `find(...)` is the modern equivalent.
- [ ] You can convert one 2.x test file to the modern `module + setupApplicationTest` shape.
- [ ] `ember-qunit@^4` is in the project's `package.json` if you plan to mix styles.

## See also

- `ember-2-classic-patterns` — the 2.x dialect under test.
- `ember-2-to-3-migration` — when to convert tests during a migration.
- Modern reference: [`ember/skills/ember-testing`](../../../ember/skills/ember-testing).
