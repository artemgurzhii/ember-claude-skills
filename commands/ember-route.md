---
description: Scaffold an Ember route (route file + template + optional controller for query params) and an application test that visits the URL.
---

# /ember-route

Scaffold a new Ember route with the route class, template, optional controller (only if query params are needed), and an application test.

## Usage

```
/ember-route <route-path> [--model=<type>] [--params=<id>] [--query=<name>:<default>]... [--auth]
```

- `<route-path>`: dotted route name. e.g. `posts.show`, `dashboard`.
- `--model=<type>`: Ember Data model to fetch (e.g. `post`). If set, the `model` hook calls `this.store.findRecord(<type>, ...)`.
- `--params=<id>`: name of the dynamic segment. e.g. `--params=post_id`. If set, `path: '/:post_id'` is added to `router.ts`.
- `--query=name:default`: declare a query param on the controller. Repeatable. Defaults to `refreshModel: true` on the route.
- `--auth`: gate with `session.requireAuthentication` in `beforeModel`.

## Steps

1. **Read** `app/router.ts` to find the right place to add the route.
2. **Add the route definition** to `Router.map(...)` preserving nesting. If `--params`, include the `path: '/:<id>'`.
3. **Generate the route file** with imports, optional `@service router`, optional `@service session`, and a `model` hook that uses Ember Data when `--model` is set.
4. **Generate the template** with an `{{outlet}}` if the route has children, otherwise a starter shell.
5. **Generate a controller** *only if* `--query` is set, with `queryParams` and `@tracked` defaults.
6. **Generate an application test** that visits the URL, asserts `currentURL()`, and (with `--auth`) asserts the unauthenticated redirect.
7. **Run Glint and the new test**, report results.

## Templates

### Route (`app/routes/<dotted/path>.ts`)

```ts
import Route from '@ember/routing/route';
import { service } from '@ember/service';
{{#if model}}
import type Store from '@ember-data/store';
import type { ModelFor<{{model}}>} from 'my-app/models/{{model}}';
{{/if}}
{{#if auth}}
import type SessionService from 'ember-simple-auth/services/session';
{{/if}}
import type RouterService from '@ember/routing/router-service';
import type Transition from '@ember/routing/transition';

interface Params {
  {{#if params}}{{params}}: string;{{/if}}
}

export default class <PascalName>Route extends Route {
  @service declare router: RouterService;
  {{#if auth}}@service declare session: SessionService;{{/if}}
  {{#if model}}@service declare store: Store;{{/if}}

  {{#if query}}
  queryParams = {
    {{#each query}}{{name}}: { refreshModel: true },{{/each}}
  };
  {{/if}}

  {{#if auth}}
  beforeModel(transition: Transition) {
    this.session.requireAuthentication(transition, 'login');
  }
  {{/if}}

  {{#if model}}
  async model({ {{params}} }: Params) {
    return this.store.findRecord('{{model}}', {{params}});
  }
  {{/if}}
}
```

### Template (`app/templates/<dotted/path>.hbs`)

```hbs
<section data-test-<dasherized>>
  {{!-- TODO: render @model --}}

  {{outlet}}
</section>
```

### Controller (only when `--query`)

```ts
// app/controllers/<dotted/path>.ts
import Controller from '@ember/controller';
import { tracked } from '@glimmer/tracking';

export default class <PascalName>Controller extends Controller {
  queryParams = [{{#each query}}'{{name}}',{{/each}}];

  {{#each query}}
  @tracked {{name}} = {{default}};
  {{/each}}
}
```

### Application test

```ts
// tests/acceptance/<dasherized>-test.ts
import { module, test } from 'qunit';
import { visit, currentURL } from '@ember/test-helpers';
import { setupApplicationTest } from 'my-app/tests/helpers';
import { setupMirage } from 'ember-cli-mirage/test-support';
import page from 'my-app/tests/pages/my-page';

module('Acceptance | <name>', function (hooks) {
  setupApplicationTest(hooks);
  setupMirage(hooks);

  test('visiting <url>', async function (assert) {
    {{#if model}}this.server.create('{{model}}', { id: '1' });{{/if}}

    await page.visit();

    assert.strictEqual(currentURL(), '/my-page');
  });

  {{#if auth}}
  test('unauthenticated visit redirects to /login', async function (assert) {
    await page.visit();

    assert.strictEqual(currentURL(), '/login');
  });
  {{/if}}
});
```

## Conventions enforced

- Query params declared on the controller, not the route.
- `refreshModel: true` set on the route's `queryParams` hash for any QP that affects the fetch.
- Route uses `RouterService.transitionTo`, never `this.transitionTo`.
- `model` hook handles 404 when `--model` is set (the agent should add the try/catch when wiring up).

## See also

- Skill: `ember-routing-and-models`
- Skill: `ember-data` (for the `findRecord` semantics)
- Skill: `ember-ecosystem-addons` → `ember-simple-auth` (for `--auth`)
