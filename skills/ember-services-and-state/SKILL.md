---
name: ember-services-and-state
description: Designing Ember services for app-wide state and side-effects, deriving state with @cached, and choosing between component state, service state, route models, and ember-data. Use when deciding "where should this state live?" or when extracting logic out of a fat component or controller.
type: reference
---

# Services & State Management

Ember has no Redux/Zustand/Pinia because the framework already gives you:

- **Singletons via DI** → services.
- **Reactivity** → `@tracked`.
- **Memoized derivations** → `@cached`.
- **Route-scoped data** → the `model` hook + Ember Data.
- **DOM-scoped data** → component fields.

Picking the right *home* for a piece of state is more important than any library choice.

## Where should this state live?

```
Is the state about a single URL's data?
  → Route model (and Ember Data).

Is the state about one component's interaction (open/closed, focused tab)?
  → Component field.

Is the state shared across multiple components, multiple routes, or persists across navigations?
  → Service.

Is it derived from other state?
  → Getter (with @cached if expensive).

Is it from the server, with caching/invalidation/relationships?
  → Ember Data (or @ember-data/request).
```

If you can't decide between component and service: start with the component, lift to a service when a second consumer appears.

## Defining a service

```ts
// app/services/notifications.ts
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';

export interface Notification {
  id: string;
  message: string;
  level: 'info' | 'warn' | 'error';
}

export default class NotificationsService extends Service {
  @tracked notifications: Notification[] = [];

  notify(message: string, level: Notification['level'] = 'info') {
    const id = crypto.randomUUID();
    this.notifications = [...this.notifications, { id, message, level }];
    setTimeout(() => this.dismiss(id), 5_000);
  }

  dismiss(id: string) {
    this.notifications = this.notifications.filter(n => n.id !== id);
  }
}

declare module '@ember/service' {
  interface Registry {
    notifications: NotificationsService;
  }
}
```

The `Registry` block is the TS pattern that makes `@service declare notifications: NotificationsService;` type-correctly throughout the app.

## Injecting a service

Anywhere with access to the owner — components, routes, controllers, helpers, modifiers, other services:

```ts
import { service } from '@ember/service';

class CheckoutComponent extends Component {
  @service declare notifications: NotificationsService;

  @action async submit() {
    try {
      await this.api.placeOrder();
      this.notifications.notify('Order placed!', 'info');
    } catch (e) {
      this.notifications.notify('Could not place order', 'error');
    }
  }
}
```

Two import paths exist: `import { service } from '@ember/service';` (modern) and `import { inject as service } from '@ember/service';` (legacy alias). Prefer the modern one.

## Built-in services worth knowing

| Service | Use for |
|---|---|
| `RouterService` (`@service('router')`) | Programmatic transitions, current URL/route name, route activity. |
| `Store` (`@service('store')`) | Ember Data root. |
| `IntlService` (`ember-intl`) | Translations + ICU formatting. |
| `SessionService` (`ember-simple-auth`) | Auth state, login/logout, token. |
| `MetricsService` (`ember-metrics`) | Analytics. |

## `@cached` — memoize a getter

A getter with `@tracked` reads is automatically reactive but **re-runs on every read by default**. For expensive derivations:

```ts
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

class TodoList {
  @tracked todos: Todo[] = [];

  @cached
  get sortedByDeadline(): Todo[] {
    // Expensive: sorts a large array
    return [...this.todos].sort((a, b) => a.deadline - b.deadline);
  }
}
```

`@cached` recomputes only when one of the tracked values it read changes. Don't apply it indiscriminately — for cheap getters it's a memory cost without payoff.

## Pattern: services for cross-route persistence

A user opens the cart on `/products/42`, navigates to `/checkout`, then back. The cart should not reload.

- Component state → lost on unmount.
- Route model → re-fetched on every entry.
- **Service** → singleton, lives for the session.

```ts
// app/services/cart.ts
export default class CartService extends Service {
  @tracked items: CartItem[] = [];

  @cached
  get subtotal(): number {
    return this.items.reduce((s, i) => s + i.price * i.quantity, 0);
  }

  add(product: Product, quantity = 1) { /* ... */ }
  remove(itemId: string) { /* ... */ }
  clear() { this.items = []; }

  hydrate(serialized: CartItem[]) { this.items = serialized; }
  serialize(): CartItem[] { return this.items; }
}
```

For persistence across reloads: hydrate from `localStorage` in the application route's `beforeModel`, and `serialize()` on changes.

## Pattern: services for ambient app state

```ts
// app/services/feature-flags.ts
import Service from '@ember/service';
import { service } from '@ember/service';
import { tracked } from '@glimmer/tracking';

export default class FeatureFlagsService extends Service {
  @service declare api: ApiService;

  @tracked private flags: Record<string, boolean> = {};

  async load() {
    this.flags = await this.api.get('/feature-flags');
  }

  isEnabled(name: string): boolean {
    return this.flags[name] ?? false;
  }
}
```

Then guard UI:

```hbs
{{#if (this.featureFlags.isEnabled "newCheckout")}}
  <NewCheckout />
{{else}}
  <LegacyCheckout />
{{/if}}
```

Bootstrap from `application` route's `beforeModel`.

## Pattern: services as orchestrators (ember-concurrency)

When the work is async with cancellation/dedupe (search, autosave, polling), a service + `ember-concurrency` task is the natural fit. See `ember-ecosystem-addons` → ember-concurrency.

```ts
import Service from '@ember/service';
import { service } from '@ember/service';
import { restartableTask } from 'ember-concurrency';

export default class SearchService extends Service {
  @service declare api: ApiService;

  searchTask = restartableTask(async (query: string) => {
    if (!query.trim()) return [];
    return this.api.get('/search', { q: query });
  });

  search(query: string) {
    return this.searchTask.perform(query);
  }
}
```

Components consume `this.search.searchTask.lastSuccessful?.value` and `this.search.searchTask.isRunning` to render results and spinners.

## Anti-patterns

| Anti-pattern | Why it's bad | Replacement |
|---|---|---|
| Putting feature logic in controllers | Controllers are singletons that survive across navigations — state leaks. | Component (UI-local) or service (cross-cutting). |
| Stuffing everything in one `app` service | Becomes a god object with no testable surface. | One service per concern: `cart`, `session`, `featureFlags`. |
| Storing tracked state in plain POJOs and mutating in place | Tracking only fires on assignment to a `@tracked` field. | `@tracked` field on a class, or `tracked-built-ins` collections. |
| Reading services through `getOwner(this).lookup(...)` from a component | Bypasses DI, hides dependencies, hard to test. | `@service declare foo: FooService;` |
| "Helper" service that wraps `Math.random` etc. | No state, no DI need. | Plain ES module function. |

## Testing services

Unit tests use `this.owner.lookup('service:notifications')`:

```ts
import { module, test } from 'qunit';
import { setupTest } from 'my-app/tests/helpers';
import type NotificationsService from 'my-app/services/notifications';

module('Unit | Service | notifications', function (hooks) {
  setupTest(hooks);

  test('notify adds a notification', function (assert) {
    const service = this.owner.lookup('service:notifications') as NotificationsService;
    service.notify('hi');
    assert.strictEqual(service.notifications.length, 1);
    assert.strictEqual(service.notifications[0].message, 'hi');
  });
});
```

Stub services in component/integration tests by registering a fake before render:

```ts
class FakeSession extends Service { @tracked isAuthenticated = true; }
this.owner.register('service:session', FakeSession);
```

See `ember-testing` for more.

## Verification

- [ ] State lives at the lowest level that needs it (component < service).
- [ ] Cross-route or "remembers between visits" state is in a service, not a controller.
- [ ] All service fields read by templates are `@tracked` (not POJOs).
- [ ] Services have a `Registry` augmentation in TS for typed injection.
- [ ] No `getOwner(this).lookup` in product code — use `@service`.
- [ ] Expensive getters are `@cached`; cheap ones aren't.

## See also

- `ember-octane-fundamentals` — `@tracked`, owners, DI basics.
- `ember-routing-and-models` — when state belongs in `model` vs a service.
- `ember-data` — when state belongs in the store rather than a custom service.
- `ember-ecosystem-addons` — `ember-concurrency` for async orchestration.
