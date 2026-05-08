---
description: Scaffold a Glimmer component (template + backing class + integration test) following Octane conventions and Polaris-friendly patterns.
---

# /ember-component

Scaffold a new Ember component with the template, backing class (TS), and a rendering test.

## Usage

```
/ember-component <component-name> [--gts] [--no-class] [--service=<name>]...
```

- `<component-name>`: dasherized name (e.g. `user-card`, `posts/list-item`).
- `--gts`: produce a single-file `.gts` (Polaris). Otherwise generates `.ts` + `.hbs`.
- `--no-class`: template-only component.
- `--service=foo`: inject a service in the backing class. Repeatable.

## Steps

1. **Confirm placement.** If the user didn't specify, ask whether the component is global (`app/components/<name>`) or namespaced (`app/components/<group>/<name>`).
2. **Detect the project's mode.** `cat tsconfig.json` and look at the `glint.environment`. If `ember-template-imports` is present, prefer `.gts` unless user said `--no-gts`.
3. **Generate files.**
   - Backing class with a typed `Signature` (Args + Element + Blocks).
   - Template using `...attributes` on the root element.
   - Integration test in `tests/integration/components/<name>-test.ts` with one render assertion.
4. **Run** `pnpm exec glint --noEmit` (or `tsc --noEmit` if Glint isn't set up) to confirm the new files type-check.
5. **Run** the new test: `pnpm test --filter '<name>'` (adjust for the project's test runner script).
6. **Report**: paste the file paths created and the result of the type-check + test run.

## Templates

### `.ts` + `.hbs` mode (Octane)

```ts
// app/components/<name>.ts
import Component from '@glimmer/component';
import { action } from '@ember/object';
{{#each services}}
import { service } from '@ember/service';
import type {{ServiceType}} from 'my-app/services/{{kebab}}';
{{/each}}

export interface <PascalName>Signature {
  Args: {
    // TODO: declare args
  };
  Element: HTMLDivElement;
  Blocks: {
    default: [];
  };
}

export default class <PascalName> extends Component<<PascalName>Signature> {
  {{#each services}}
  @service declare {{name}}: {{ServiceType}};
  {{/each}}
}
```

```hbs
{{! app/components/<name>.hbs }}
<div ...attributes>
  {{yield}}
</div>
```

### `.gts` mode (Polaris)

```ts
// app/components/<name>.gts
import Component from '@glimmer/component';

export interface <PascalName>Signature {
  Args: {};
  Element: HTMLDivElement;
  Blocks: { default: [] };
}

export default class <PascalName> extends Component<<PascalName>Signature> {
  <template>
    <div ...attributes>
      {{yield}}
    </div>
  </template>
}
```

### Test (always generated)

```ts
// tests/integration/components/<name>-test.ts
import { module, test } from 'qunit';
import { setupRenderingTest } from 'my-app/tests/helpers';
import { render } from '@ember/test-helpers';
import { hbs } from 'ember-cli-htmlbars';

module('Integration | Component | <name>', function (hooks) {
  setupRenderingTest(hooks);

  test('renders its block content', async function (assert) {
    await render(hbs`<<PascalName>>hello</<PascalName>>`);
    assert.dom().hasText('hello');
  });
});
```

## Conventions enforced

- `@glimmer/component`, never `@ember/component`.
- `...attributes` placed last on the root element.
- `Signature` typed with `Args`, `Element`, `Blocks`.
- `data-test-*` attributes added by default to interactive elements (button, input, select).
- Test imports from `'my-app/tests/helpers'` (the local re-export of `ember-qunit`).

## See also

- Skill: `ember-components-and-templates`
- Skill: `ember-octane-fundamentals`
- Skill: `ember-polaris-migration` (when `--gts`)
