---
name: ember-routing-and-models
description: Ember Router — route definitions, route hooks (beforeModel, model, afterModel, redirect), nested routes, dynamic segments, query params, transitions, error/loading substates, and LinkTo. Use when adding URLs, fetching route-level data, handling auth redirects, or debugging "why isn't this route entering."
type: reference
---

# Ember Routing & Route-Level Data

The router is the entry point for every URL in an Ember app. It also owns **route-level data fetching** — the `model` hook is where you load data tied to a URL, and the route is the natural place to handle redirects, auth, and not-found.

## Defining the URL map

```ts
// app/router.ts
import EmberRouter from '@ember/routing/router';
import config from 'my-app/config/environment';

export default class Router extends EmberRouter {
  location = config.locationType;
  rootURL = config.rootURL;
}

Router.map(function () {
  this.route('login');
  this.route('dashboard');

  this.route('posts', function () {
    this.route('new');
    this.route('show', { path: '/:post_id' }, function () {
      this.route('edit');
      this.route('comments');
    });
  });

  this.route('not-found', { path: '/*path' });
});
```

Each `this.route(...)` becomes:
- A **route file** at `app/routes/<name>.ts`.
- A **template** at `app/templates/<name>.hbs` (or colocated under `app/routes/...` in newer setups; both work).
- An optional **controller** at `app/controllers/<name>.ts`.

For the nested example above, `posts.show` lives at `app/routes/posts/show.ts` and `app/templates/posts/show.hbs`. Its parent template (`posts.hbs`) must include `{{outlet}}` for nested routes to render.

## Route hooks (in order)

```
URL change
   │
   ▼
beforeModel(transition)   ← redirect / auth gate / preload
   │
   ▼
model(params, transition) ← fetch the data this URL is "about"
   │
   ▼
afterModel(model, transition) ← post-fetch checks (404, permission)
   │
   ▼
redirect(model, transition)   ← optional final redirect
   │
   ▼
setupController(controller, model)
   │
   ▼
Template renders
```

If any hook calls `transition.abort()` or `this.router.transitionTo(...)`, the rest are skipped.

## The `model` hook

```ts
// app/routes/posts/show.ts
import Route from '@ember/routing/route';
import { service } from '@ember/service';
import type Store from '@ember-data/store';
import type RouterService from '@ember/routing/router-service';

interface Params { post_id: string }

export default class PostsShowRoute extends Route<Promise<Post>> {
  @service declare store: Store;
  @service declare router: RouterService;

  async model({ post_id }: Params) {
    try {
      return await this.store.findRecord('post', post_id);
    } catch (err) {
      if ((err as { errors?: { status?: string }[] }).errors?.[0]?.status === '404') {
        this.router.replaceWith('not-found');
        return;
      }
      throw err;
    }
  }
}
```

Key points:
- The `model` hook receives `params` (route's dynamic segments) and `transition`.
- Return the data — promises are awaited automatically before the route resolves.
- The **resolved value** is available as `@model` in the template and as `this.model` on the controller.

## Dynamic segments

```ts
this.route('posts', function () {
  this.route('show', { path: '/:post_id' });
});
```

- `:post_id` is a dynamic segment.
- It arrives as `params.post_id` (always a string).
- For Ember Data models, `serialize`/`model` convention auto-fetches via `findRecord`. For non–Ember Data, do the lookup yourself.
- `*wildcard` matches the rest of the URL — useful for catch-all 404 routes.

## `<LinkTo>` and transitions

```hbs
<LinkTo @route="posts.show" @model={{@post}}>{{@post.title}}</LinkTo>

<LinkTo @route="posts.show" @models={{array @post.id}}>By ID</LinkTo>

<LinkTo @route="posts" @query={{hash sort="recent"}}>Recent</LinkTo>
```

Programmatically:

```ts
@service declare router: RouterService;

this.router.transitionTo('posts.show', post.id, { queryParams: { tab: 'comments' } });
this.router.replaceWith('login');           // doesn't add a history entry
this.router.refresh();                      // re-runs current route's hooks
```

`@model` accepts a single dynamic-segment value; `@models` accepts an array for nested dynamic segments.

## Query params

Defined on the **controller**, not the route:

```ts
// app/controllers/posts/index.ts
import Controller from '@ember/controller';
import { tracked } from '@glimmer/tracking';

export default class PostsIndexController extends Controller {
  queryParams = ['sort', 'page'];

  @tracked sort = 'recent';
  @tracked page = 1;
}
```

```hbs
{{! app/templates/posts.hbs }}
<select {{on "change" this.changeSort}}>
  <option value="recent" selected={{eq this.sort "recent"}}>Recent</option>
  <option value="popular" selected={{eq this.sort "popular"}}>Popular</option>
</select>
{{outlet}}
```

To re-run `model` when QPs change:

```ts
// app/routes/posts/index.ts
import Route from '@ember/routing/route';

export default class PostsIndexRoute extends Route {
  queryParams = {
    sort: { refreshModel: true },
    page: { refreshModel: true },
  };

  async model({ sort, page }: { sort: string; page: number }) {
    return this.store.query('post', { sort, page });
  }
}
```

Without `refreshModel: true`, query param changes update the URL but do not re-run `model`.

> **Polaris note:** RFC #1018 moves query params to a service-based API (`@queryParam`). Until that lands, the controller-based pattern above is canonical.

## Auth and redirects (`beforeModel`)

```ts
import Route from '@ember/routing/route';
import { service } from '@ember/service';
import type SessionService from 'ember-simple-auth/services/session';

export default class DashboardRoute extends Route {
  @service declare session: SessionService;

  beforeModel(transition: Transition) {
    this.session.requireAuthentication(transition, 'login');
  }
}
```

Or roll your own:

```ts
beforeModel(transition: Transition) {
  if (!this.session.isAuthenticated) {
    this.session.attemptedTransition = transition;
    this.router.transitionTo('login');
  }
}
```

For role-based access, prefer `afterModel` so you have the resolved record:

```ts
afterModel(post: Post) {
  if (!this.currentUser.canEdit(post)) {
    this.router.replaceWith('posts.show', post);
  }
}
```

## Loading and error substates

Convention-driven:

| File | Renders when... |
|---|---|
| `app/routes/posts/loading.ts` + `loading.hbs` | The sibling/parent route is mid-transition (any hook returns a pending promise). |
| `app/routes/posts/error.ts` + `error.hbs` | A hook throws or rejects. |
| `app/templates/loading.hbs` | Application-wide loading fallback. |
| `app/templates/error.hbs` | Application-wide error fallback. |

The error template gets the error as `@model`. Don't over-rely on these; they're for *route-level* failures.

## Multiple parallel models

If a route needs several fetches, return an `RSVP.hash`:

```ts
import RSVP from 'rsvp';

async model() {
  return RSVP.hash({
    user: this.store.findRecord('user', 'me'),
    notifications: this.store.query('notification', { unread: true }),
    settings: fetch('/api/settings').then(r => r.json()),
  });
}
```

The template gets `@model.user`, `@model.notifications`, etc.

## Controllers — when you need them

Controllers are still required for query params, but for most routes you don't need one. Today's recommendation:

- Use a controller **only** for query params or template-level computed state.
- Put feature logic in **services** or **route classes**, not controllers.
- Avoid stuffing actions into controllers — push them into components.

Controllers are singletons that survive across route transitions; this can lead to subtle state leaks. When in doubt, prefer a component.

## Transitions — pause, abort, retry

```ts
async beforeModel(transition: Transition) {
  if (this.cart.hasUnsavedChanges) {
    if (!window.confirm('Discard changes?')) {
      transition.abort();
    }
  }
}
```

You can also store and **retry** a transition (typical for the login flow):

```ts
// during login redirect
this.session.attemptedTransition = transition;
this.router.transitionTo('login');

// after successful login
const attempted = this.session.attemptedTransition;
this.session.attemptedTransition = null;
attempted ? attempted.retry() : this.router.transitionTo('dashboard');
```

## Router service — what to read where

```ts
@service declare router: RouterService;

this.router.currentRouteName     // 'posts.show'
this.router.currentURL           // '/posts/42?tab=comments'
this.router.urlFor('posts.show', 42)
this.router.isActive('posts')    // true if current route starts with 'posts'
this.router.on('routeDidChange', (transition) => { ... });
```

Don't reach into `getOwner(this).lookup('router:main')` — use `RouterService`.

## Performance: lazy + parallel fetches

- Hooks at the same nesting level run **in parallel** with each other across siblings only after the parent resolves.
- Parent routes block children. Don't put slow optional data in `application` route's `model`.
- For "show the page now, fetch optional data later," start the fetch in `model`, return a fast piece, and stash the slow promise on a service or pass it as a `Promise<X>` arg the component can `await`.
- `ember-data`'s store de-dupes parallel `findRecord` calls for the same id.

## Common mistakes

| Mistake | Fix |
|---|---|
| Forgetting `{{outlet}}` in a parent template | Children render but you don't see them. |
| Mutating params or model in the route after `setupController` | The model is reactive via Ember Data or `@tracked`; mutate the record, not the route. |
| Putting auth in `model` instead of `beforeModel` | The fetch happens before redirect — wastes a request and may 401-error. |
| Using `replaceWith` from inside a `model` hook | Use `beforeModel`/`afterModel`. From `model`, return a rejected promise or call `transition.abort()` then `transitionTo`. |
| Defining `queryParams` on the route only | They must be declared on the controller; `queryParams` on the route is just for `refreshModel` flags. |
| `this.transitionTo` (deprecated on Route) | Inject `RouterService` and use `this.router.transitionTo`. |

## Verification

- [ ] Each dynamic segment maps to a real fetch or a stable lookup.
- [ ] Auth gates live in `beforeModel`, permission checks in `afterModel`.
- [ ] Query params declared on the controller; `refreshModel: true` set if model depends on them.
- [ ] `{{outlet}}` present in parent templates.
- [ ] No `transitionTo` on the Route — use `RouterService`.
- [ ] Loading/error substates exist for slow or failure-prone routes.

## See also

- `ember-data` — what `findRecord` / `query` / `peekRecord` actually do, and the new `@ember-data/request` builders.
- `ember-services-and-state` — extracting non-route data fetching out of routes.
- `ember-ecosystem-addons` — `ember-simple-auth` for auth-gated routes.
