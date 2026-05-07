---
name: ember-testing
description: Testing Ember apps — QUnit + @ember/test-helpers, the unit/integration/application split, ember-test-selectors, ember-cli-page-object, ember-cli-mirage, accessibility checks, and async settledness. Use when writing or fixing tests, designing a test strategy, or debugging flaky/non-settled tests.
type: reference
---

# Ember Testing

Ember has the strongest test story of any major frontend framework. Use it.

## The three test types

| Type | Helper | Boots... | Use for |
|---|---|---|---|
| **Unit** | `setupTest(hooks)` | Container only — no DOM, no router. | Services, models, utilities. |
| **Rendering** (a.k.a. integration) | `setupRenderingTest(hooks)` | Container + a render context (`render(hbs\`...\`)`). | Components, helpers, modifiers. |
| **Application** (a.k.a. acceptance) | `setupApplicationTest(hooks)` | Full app — router boots, services real, DOM live. | User flows, route-level behavior. |

Default to **rendering tests** for components — they're fast and exercise the real template/component contract. Save application tests for end-to-end flows.

## Anatomy of a test file

```ts
// tests/integration/components/user-card-test.ts
import { module, test } from 'qunit';
import { setupRenderingTest } from 'my-app/tests/helpers';
import { render, click, fillIn, settled } from '@ember/test-helpers';
import { hbs } from 'ember-cli-htmlbars';

module('Integration | Component | user-card', function (hooks) {
  setupRenderingTest(hooks);

  test('renders the user name', async function (assert) {
    this.user = { name: 'Ada Lovelace', email: 'ada@example.com' };

    await render(hbs`<UserCard @user={{this.user}} />`);

    assert.dom('[data-test-name]').hasText('Ada Lovelace');
    assert.dom('[data-test-email]').hasText('ada@example.com');
  });
});
```

`tests/helpers.ts` re-exports the helpers from `ember-qunit` plus any project-specific setup (Mirage, intl, etc.).

## `@ember/test-helpers` — the standard library

| Helper | What it does |
|---|---|
| `render(hbs\`...\`)` | Render a template and return when settled. Rendering tests only. |
| `visit(url)` | Visit a URL through the router. Application tests only. |
| `currentURL()` / `currentRouteName()` | Read router state. |
| `click(selector)` | Click and wait for settled. |
| `fillIn(selector, value)` | Set value + dispatch input/change. |
| `typeIn(selector, value)` | Type one char at a time (for keypress handlers). |
| `select(selector, value, multiple?)` | `<select>`. |
| `triggerEvent(selector, eventType, options?)` | Custom events. |
| `triggerKeyEvent(selector, eventType, key)` | Keyboard. |
| `find(selector)` / `findAll(selector)` | DOM lookup. |
| `settled()` | Wait until: no pending promises, no scheduled runs, no pending requests, no test waiters. |
| `waitFor(selector, { timeout, count })` | Wait until selector is present. |
| `waitUntil(predicate)` | Wait until predicate returns truthy. |

**Always `await` user-action helpers.** They handle settling for you. Don't add manual `setTimeout`s.

## Selectors — use `data-test-*` via `ember-test-selectors`

Hard-coded class selectors break when designers refactor CSS. Use stable test selectors:

```hbs
<button data-test-save type="button" {{on "click" this.save}}>Save</button>
```

Install [`ember-test-selectors`](https://github.com/simplabs/ember-test-selectors) and the `data-test-*` attributes are **auto-stripped from production builds**. You get clean test selectors with zero runtime cost.

```ts
await click('[data-test-save]');
assert.dom('[data-test-save]').isDisabled();
```

`qunit-dom` (the `assert.dom(...)` API) is bundled with `ember-qunit`. Use it for DOM assertions — `hasText`, `hasClass`, `hasAttribute`, `isDisabled`, `isVisible`, etc.

## Page objects — `ember-cli-page-object`

For non-trivial components and pages, encapsulate selectors in a page object so test bodies read like behavior, not DOM lookups:

```ts
// tests/pages/user-card.ts
import { create, clickable, text, isPresent } from 'ember-cli-page-object';

export default create({
  scope: '[data-test-user-card]',
  name: text('[data-test-name]'),
  email: text('[data-test-email]'),
  hasEditButton: isPresent('[data-test-edit]'),
  edit: clickable('[data-test-edit]'),
});
```

```ts
import page from '../pages/user-card';

test('clicking edit opens the editor', async function (assert) {
  this.user = { name: 'Ada' };
  await render(hbs`<UserCard @user={{this.user}} />`);

  assert.strictEqual(page.name, 'Ada');
  await page.edit();
  assert.dom('[data-test-editor]').exists();
});
```

The payoff is huge for application tests with multi-screen flows. Sticky rule: a page object never asserts — it only describes the page surface. Assertions live in the test.

## Mocking the API — `ember-cli-mirage`

[`ember-cli-mirage`](https://www.ember-cli-mirage.com/) gives you an in-browser fake server with a database, factories, scenarios, and route handlers. Use it for development *and* tests.

```ts
// mirage/config.ts
export default function () {
  this.namespace = '/api';

  this.get('/posts');
  this.get('/posts/:id');
  this.post('/posts');
  this.patch('/posts/:id');
  this.del('/posts/:id');

  this.get('/posts/:id/comments', (schema, request) => {
    const post = schema.posts.find(request.params.id);
    return post.comments;
  });
}
```

```ts
// mirage/factories/post.ts
import { Factory } from 'miragejs';

export default Factory.extend({
  title(i: number) { return `Post ${i}`; },
  body() { return 'lorem ipsum'; },
  publishedAt: () => new Date(),
});
```

In tests:

```ts
import { setupMirage } from 'ember-cli-mirage/test-support';

module('Application | posts', function (hooks) {
  setupApplicationTest(hooks);
  setupMirage(hooks);

  test('lists posts', async function (assert) {
    this.server.createList('post', 3);

    await visit('/posts');

    assert.dom('[data-test-post-row]').exists({ count: 3 });
  });
});
```

Mirage models the JSON:API conventions Ember Data expects, so most setups Just Work without serializer overrides.

## Testing services — unit tests with `setupTest`

```ts
import { module, test } from 'qunit';
import { setupTest } from 'my-app/tests/helpers';

module('Unit | Service | cart', function (hooks) {
  setupTest(hooks);

  test('subtotal sums item prices', function (assert) {
    const cart = this.owner.lookup('service:cart') as CartService;
    cart.add({ id: '1', price: 10, quantity: 2 });
    cart.add({ id: '2', price: 5, quantity: 1 });
    assert.strictEqual(cart.subtotal, 25);
  });
});
```

To stub a dependency, register a fake before lookup:

```ts
class FakeApi extends Service {
  get = sinon.stub().resolves([]);
}

hooks.beforeEach(function () {
  this.owner.register('service:api', FakeApi);
});
```

## Testing components that depend on services

Same pattern — register a fake service in the test:

```ts
test('shows error toast on failed save', async function (assert) {
  class FakeNotifications extends Service {
    notify = sinon.spy();
  }
  this.owner.register('service:notifications', FakeNotifications);
  const notifications = this.owner.lookup('service:notifications') as FakeNotifications;

  await render(hbs`<SaveButton />`);
  await click('[data-test-save]');

  assert.true(notifications.notify.calledWith('Could not save', 'error'));
});
```

## Testing routes

For a route's `model`/`beforeModel`/`afterModel` logic, write an **application test** that visits the URL and asserts on what rendered or where the user was redirected:

```ts
test('unauthenticated visit to /dashboard redirects to /login', async function (assert) {
  await visit('/dashboard');
  assert.strictEqual(currentURL(), '/login');
});
```

Don't unit-test routes — they're glue that's only meaningful when the router runs them.

## Async + settled — the only flake source you should ever face

`await render(...)`, `await click(...)`, etc. all return only after Ember considers the system **settled**:

- All scheduled runs are flushed.
- All pending promises resolved.
- No pending AJAX (via test waiters).
- No outstanding `requestAnimationFrame`.

If you have flaky tests, the cause is almost always **work that isn't registered with a test waiter**. Fixes:

- Use `ember-concurrency` tasks — they integrate with test waiters automatically.
- Wrap raw timers with [`@ember/test-waiters`](https://github.com/emberjs/ember-test-waiters):

```ts
import { waitForPromise } from '@ember/test-waiters';

@action async loadStuff() {
  await waitForPromise(somePromise);
}
```

- Or simply `await` the work in your component action.

For tricky cases use `await waitFor('[data-test-loaded]')` or `await waitUntil(() => something)`.

## Accessibility tests — `ember-a11y-testing`

```ts
import a11yAudit from 'ember-a11y-testing/test-support/audit';

test('home page is accessible', async function (assert) {
  await visit('/');
  await a11yAudit();
  assert.ok(true, 'no a11y errors');
});
```

`ember-a11y-testing` runs `axe-core` and fails the test on serious violations. Integrate it into representative pages and key components.

## Snapshot / visual regression

Avoid full-DOM snapshots — they're noisy. For visual checks, use Percy (`@percy/ember`) on a small set of representative pages, or screenshot-diff via Playwright in a separate suite.

## Test data — keep it minimal and named

```ts
// Bad: opaque, breaks when defaults change
this.user = { id: 1, name: 'a', email: 'a@b.c', isAdmin: true, joinedAt: new Date() };

// Good: only what the test needs
this.user = { id: 1, name: 'Ada', isAdmin: true };
```

For Mirage, keep factories minimal and use traits (`Factory.extend({ traits: { admin: trait({ role: 'admin' }) } })`) for variants.

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| Hard-coding class selectors that change with CSS | `data-test-*` via `ember-test-selectors`. |
| `await new Promise(r => setTimeout(r, 100))` | `await settled()` or `await waitFor(...)`. |
| Mocking the entire store in unit tests | Use `setupTest` and the real store; stub fetches via Mirage. |
| Application tests that fill in many forms | Split into focused rendering tests; keep applications for the happy paths. |
| One mega `module` with shared `let user;` | Each test sets up its own state — DAMP > DRY in tests. |
| `assert.equal(x, true)` | `assert.true(x)` / `assert.dom('...').isChecked()` etc. — clearer failures. |

## Verification

- [ ] Components have a rendering test for the happy path and major branches.
- [ ] Services have unit tests for each public method.
- [ ] Routes are covered by application tests asserting URL + content.
- [ ] Selectors are `data-test-*`; production builds strip them via `ember-test-selectors`.
- [ ] Mirage handles every endpoint the test suite touches.
- [ ] No raw `setTimeout` or polling in tests; all waits go through `@ember/test-helpers`.
- [ ] At least one a11y audit per major page.

## See also

- `ember-ecosystem-addons` — Mirage, page-object, test-selectors, a11y-testing, ember-concurrency.
- `ember-octane-fundamentals` — `@tracked` reactivity (impacts what your test sees).
- `ember-routing-and-models` — application-test redirects and route hooks.
