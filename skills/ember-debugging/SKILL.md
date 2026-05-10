---
name: ember-debugging
description: Debugging Ember apps — Ember Inspector panels, deprecation source tracing with the deprecation workflow, autotracking re-render audits, runloop and settled-state diagnosis. Use when chasing "why is this re-rendering," tracing a deprecation back to its caller, or fixing a flaky non-settled test.
type: reference
---

# Ember Debugging

Ember has more built-in introspection than any other major frontend framework. The trick is knowing which tool answers which question.

## The triage table

| Symptom | First tool | Then |
|---|---|---|
| "Why is this component re-rendering?" | Ember Inspector → **Render Performance** | Audit `@tracked` reads in the getter chain. |
| "Where does this deprecation come from?" | `ember-cli-deprecation-workflow` config + browser console source link | Patch the offending caller; bump workflow to `silence`. |
| "Test ended before all promises settled" | `await settled()` and remove orphan `setTimeout` | Move work behind `@ember/test-waiters`. |
| "Service injection returned undefined" | Ember Inspector → **Container** panel | Check `Registry` declaration + filename casing. |
| "Route transition silently aborted" | Ember Inspector → **Routes** + browser network panel | Look for `transition.abort()`, `redirect()`, or rejected `model`. |
| "Promise rejection swallowed" | Ember Inspector → **Promises** panel | Add `.catch` or surface via `notifications` service. |

## Ember Inspector

Browser extension for Chrome and Firefox. Install once; it auto-attaches to any page running Ember.

| Panel | Best at |
|---|---|
| **Component Tree** | Inspecting the live render hierarchy, args, and tracked state of any rendered Glimmer component. |
| **Routes** | Confirming which route is active, what the URL maps to, what `model` returned. |
| **Data** | Browsing the Ember Data store — every record, attribute, and relationship state (`isLoaded`, `isDirty`, `isError`). |
| **Render Performance** | Profiling re-renders — see the count and duration of each component's renders during a recorded interaction. |
| **Promises** | Tracing pending/fulfilled/rejected promises with their stack traces. Single best tool for "why is the spinner stuck." |
| **Container** | Listing every registered service, controller, route, helper, modifier — confirms a registration actually exists. |
| **Deprecations** | Aggregated deprecation log with source links. |

To inspect any object from the console:

```js
// $E is the Ember Inspector global — last selected object in any panel.
$E.someTrackedField = 'new value'; // mutates and triggers a re-render
```

## Deprecation tracing

### Read the deprecation, don't suppress it

Each deprecation message carries an `id` and a `until` version. The url at the bottom of every message links to deprecations.emberjs.com with the migration recipe.

```
DEPRECATION: Using `Ember.set` is deprecated. Please use `set` from `@ember/object`.
[deprecation id: ember-source.ember-set]
For more details see: https://deprecations.emberjs.com/v5.x/#ember-source-ember-set
```

The `id` is the only stable identifier. The wording changes between versions.

### The deprecation workflow

`ember-cli-deprecation-workflow` lets you triage deprecations once, then keep the build green while you fix them:

```js
// config/deprecation-workflow.js
self.deprecationWorkflow = self.deprecationWorkflow || {};
self.deprecationWorkflow.config = {
  workflow: [
    { handler: 'silence', matchId: 'ember-source.ember-set' },
    { handler: 'throw',   matchId: 'ember-data.legacy-relationship-options' },
    { handler: 'log',     matchId: 'ember-source.implicit-injections' },
  ],
};
```

- `silence` — known, scheduled to fix; don't spam the console.
- `log` — surface in the console while you investigate.
- `throw` — fail the build/tests on this one. Use this for things you've fixed and want to lock down.

### Find the source

Modern Ember deprecations include the call site in their stack frame. In Chrome:

1. Open DevTools console.
2. Right-click the stack frame inside the deprecation message → **Show function definition**.
3. The link jumps to your source file (works through source maps).

If the source is inside an addon, that's a hint: the addon has not yet adopted the new API and may need an update or a replacement (see `ember-ecosystem-addons`).

## Autotracking re-render audits

A re-render in Octane fires when a `@tracked` value that the template *read* changes. The most common surprises:

1. **A POJO mutation** — `this.user.name = 'x'` does not trigger a re-render unless `name` is `@tracked` on `User`. Wrap with `tracked-built-ins` or assign a new object.
2. **An array push** — `this.items.push(x)` does not trigger; `this.items = [...this.items, x]` does. Or use `tracked-built-ins`'s `TrackedArray`.
3. **A getter without `@cached`** — re-runs on every read. If the getter touches a slow computation, every re-render pays the cost. Add `@cached` and confirm with the **Render Performance** panel.

### Locate the culprit

```js
// In the browser console, with Ember Inspector loaded:
import { tagFor } from '@glimmer/validator';

// Pick the field you suspect.
let tag = tagFor(this.someService, 'someField');
console.log(tag.value());

// Mutate and watch the value change. If it doesn't, you found a non-reactive write.
this.someService.someField = 'new';
console.log(tag.value());
```

In tests, wrap the suspect code in `await settled()` and assert against `assert.dom(...)` — if the DOM doesn't reflect the change, the write isn't reactive.

## Settled state and async

`@ember/test-helpers` exposes `settled()` and `isSettled()`. A test that "ends before everything settled" usually means a promise was started but not registered as a waiter.

### `await settled()`

`settled()` resolves when:

- The runloop is empty.
- All `@ember/test-waiters` waiters report `false`.
- All pending route transitions have completed.
- All AJAX requests have resolved.
- All Ember Data store operations have finished.

```ts
import { settled } from '@ember/test-helpers';

test('async cart update', async function (assert) {
  this.cart.add(product);
  await settled();
  assert.dom('[data-test-cart-count]').hasText('1');
});
```

### `@ember/test-waiters` for arbitrary work

If you have async work that's not an AJAX call, runloop task, or transition, register it as a waiter so `settled()` knows about it:

```ts
import { waitForPromise } from '@ember/test-waiters';

class SearchService extends Service {
  async run(query: string) {
    return waitForPromise(this.api.get(`/search?q=${query}`));
  }
}
```

Without `waitForPromise`, the test framework will think the work is done before the promise resolves and assertions race the response.

### Diagnosing a hang

A test that hangs forever (rather than fails fast) almost always means a waiter never resolves. Check:

1. Is there a `setTimeout` not wrapped in `waitForPromise`? It will hang forever in tests because the test framework doesn't `await` it.
2. Is there an `ember-concurrency` task that loops forever (`while (true) yield timeout(1000)`)?
3. Is there a router transition that's `pause`d?

Set `Testem.afterTests` or add `console.log(isSettled())` in the suspect code path to print which waiter is still pending.

## Runloop debugging

You rarely need to think about the runloop in Octane code. When you do:

| API | Use for |
|---|---|
| `schedule('afterRender', this, fn)` | Run code after the current render flush — DOM is up to date. |
| `next(this, fn)` | Defer to the next runloop turn (microtask-ish). |
| `cancel(token)` | Cancel a previously scheduled callback. |

In tests, *never* use `setTimeout`. Use `waitFor` from `@ember/test-helpers`:

```ts
import { waitFor } from '@ember/test-helpers';

await waitFor('[data-test-toast]'); // resolves when the element appears, or fails after a timeout.
```

## Common bug shapes

### "It works on first navigation but not the second"

Likely a controller or service holding stale state. Controllers in Ember are *singletons* — they persist across visits to the same route. Reset state in `setupController` or move it out of the controller.

### "It works in dev but the test sees null"

Likely route timing. The test ran the assertion before the `model` hook resolved. Use `await visit('/route')` (which waits for `settled`) and assert after. If you're using `setupApplicationTest`, `visit` already awaits — check that you're not chaining `then(...)` instead of `await`.

### "The service has the right state but the template renders the old one"

The template read a non-tracked field, or read a tracked field through a non-tracked path. Audit every `this.foo.bar.baz` chain — every link must be tracked or a stable reference.

### "ember-data record won't update in the UI"

Check `record.isDirty` in the **Data** panel. If the field changed but `isDirty` is false, you mutated a relationship array in place rather than calling `set` or replacing it. Use `record.someRelationship.pushObject(x)` or assign a new array.

### "Production build broken, dev fine"

Usually a missing addon depending-on-itself, an Embroider compat issue, or a template-only component the AOT compiler couldn't resolve. Build with `EMBROIDER_DEBUG=1` (Embroider) or `ember build --environment=production --output-path=tmp/prod` and inspect the failure locally.

## Verification

- [ ] Ember Inspector is installed in your dev browser.
- [ ] `ember-cli-deprecation-workflow` is configured for any non-trivial app — every deprecation has an explicit handler.
- [ ] Every async operation outside ember-data / runloop / router is wrapped in `waitForPromise` or behind an `ember-concurrency` task.
- [ ] No `setTimeout` in product code without a corresponding `waitFor` strategy in tests.
- [ ] Tests use `await visit/click/fillIn` and `await settled()`, never `then(...)` chains.
- [ ] Re-render hot paths confirmed clean via the **Render Performance** panel before optimizing.

## See also

- `ember-octane-fundamentals` — what `@tracked` actually does.
- `ember-services-and-state` — `@cached`, where state belongs.
- `ember-testing` — settled state, page objects, Mirage.
- `ember-ecosystem-addons` → `observability.md` — Sentry/Bugsnag for production-side debugging.
