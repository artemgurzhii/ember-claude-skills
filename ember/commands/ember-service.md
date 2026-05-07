---
description: Scaffold an Ember service with TypeScript types, Registry augmentation, and a unit test.
---

# /ember-service

Scaffold an Ember service with `@tracked` state, the `@ember/service` `Registry` augmentation (so `@service declare foo: FooService;` is type-correct app-wide), and a unit test.

## Usage

```
/ember-service <name> [--inject=<service>]...
```

- `<name>`: dasherized service name (e.g. `shopping-cart`, `feature-flags`).
- `--inject=<service>`: another service to inject. Repeatable.

## Steps

1. **Generate the service file** at `app/services/<name>.ts`.
2. **Add the `Registry` augmentation** at the bottom of the same file.
3. **Generate the unit test** at `tests/unit/services/<name>-test.ts`.
4. **Run** `pnpm exec glint --noEmit` and the new test, report results.

## Templates

### Service (`app/services/<name>.ts`)

```ts
import Service from '@ember/service';
import { service } from '@ember/service';
import { tracked } from '@glimmer/tracking';
{{#each injects}}
import type {{TypeName}}Service from 'my-app/services/{{name}}';
{{/each}}

export default class <PascalName>Service extends Service {
  {{#each injects}}
  @service declare {{name}}: {{TypeName}}Service;
  {{/each}}

  // TODO: declare @tracked state and public API here.
}

declare module '@ember/service' {
  interface Registry {
    '<dasherized-name>': <PascalName>Service;
  }
}
```

### Unit test (`tests/unit/services/<name>-test.ts`)

```ts
import { module, test } from 'qunit';
import { setupTest } from 'my-app/tests/helpers';
import type <PascalName>Service from 'my-app/services/<name>';

module('Unit | Service | <name>', function (hooks) {
  setupTest(hooks);

  test('it exists', function (assert) {
    const service = this.owner.lookup('service:<name>') as <PascalName>Service;
    assert.ok(service);
  });
});
```

## Conventions enforced

- `Registry` augmentation in the same file as the service — so importers get typed injection automatically.
- Every public field that templates read is `@tracked`.
- Unit test uses `this.owner.lookup`, not direct instantiation.
- TS uses `declare` on `@service`-decorated fields.

## See also

- Skill: `ember-services-and-state`
- Skill: `ember-typescript-and-glint` (Registry pattern)
- Skill: `ember-octane-fundamentals` (DI basics)
