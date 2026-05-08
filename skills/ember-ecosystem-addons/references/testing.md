# Testing

## [`ember-test-selectors`](https://github.com/simplabs/ember-test-selectors)

Strips `data-test-*` attributes from production builds. Test the contract without polluting prod HTML.

```bash
ember install ember-test-selectors
```

```hbs
<button data-test-save type="button">Save</button>
```

## [`ember-cli-page-object`](https://ember-cli-page-object.js.org)

Page-object pattern for tests. Encapsulates DOM selectors so test bodies read like behavior, not lookups.

```bash
ember install ember-cli-page-object
```

See `ember-testing` for a full example.

## [`ember-cli-mirage`](https://www.ember-cli-mirage.com)

In-browser fake API server with a database, factories, scenarios, and route handlers. Use for both dev (`mirage/scenarios/default.ts`) and tests.

```bash
ember install ember-cli-mirage
```

Caveat: release cadence has slowed (last publish 2024-09 as of 2026-05). It still works on modern Ember and is the largest install base, but new projects increasingly reach for [MSW](https://mswjs.io) instead — same ergonomics with cross-framework support and active development. Stay on Mirage if you already have a Mirage server with factories and scenarios; consider MSW for greenfield apps. Note: you can also use `miragejs` directly.

## [`ember-a11y-testing`](https://github.com/ember-a11y/ember-a11y-testing)

Wraps axe-core for in-test a11y audits.

```bash
ember install ember-a11y-testing
```

```ts
import a11yAudit from 'ember-a11y-testing/test-support/audit';
import page from 'my-app/tests/pages/my-page';

await page.visit();
await a11yAudit();
```

## [`@percy/ember`](https://github.com/percy/percy-ember)

Visual regression via Percy. Install only if you have a Percy account.

## Acceptance test example

```hbs
<!-- app/templates/application.hbs -->

<HeadlessForm data-test-form as |form|>
  <form.Field @name='name' as |field|>
    <div data-test-form-name>
      <field.Label>Name</field.Label>
      <field.Input data-test-form-name-input />
      <field.Errors data-test-form-name-errors />
    </div>
  </form.Field>

  <button type='submit' disabled={{form.isInvalid}} data-test-form-submit>Submit</button>
</HeadlessForm>
```

```ts
// tests/pages/application.ts

import {
  create,
  visitable,
  fillable,
  clickable,
  property,
  isPresent,
  text,
} from 'ember-cli-page-object';

export default create({
  visit: visitable('/'),

  form: {
    scope: '[data-test-form]',

    name: {
      fill: fillable('[data-test-form-name-input]'),

      errors: {
        scope: '[data-test-form-name-errors]',

        text: text(),
        isPresent: isPresent(),
      },
    },

    submit: {
      scope: '[data-test-form-submit]',

      isDisabled: property('disabled'),

      click: clickable(),
    },
  },
});

```

```ts
// tests/acceptance/application-test.ts

module('Acceptance | application', function (hooks) {
  setupApplicationTest(hooks, {});

  test('form works correctly', async function (assert) {
    assert.expect(8);

    await page.visit();

    assert.strictEqual(
      currentURL(),
      '/',
      'root page is opened',
    );

    await snapshot('/application: edit form is shown');

    await page.form.name.fill('');
    await page.form.submit.click();

    assert.ok(
      page.form.name.errors.isPresent,
      'form name input error section is shown',
    );

    assert.strictEqual(
      page.form.name.errors.text,
      'Name is required', // or take it from translation file
      'correct validation error message is shown',
    );

    assert.ok(
      page.form.submit.isDisabled,
      'submit button is marked as disabled when form is invalid',
    );

    await page.form.name.fill('lorem');

    assert.notOk(
      page.form.submit.isDisabled,
      'submit button is enabled when form is valid',
    );

    window.server.patch(
      '/users/:id',
      function ({ users }, { requestBody }) {
        const {
          data: { id, attributes },
        } = JSON.parse(requestBody);

        assert.strictEqual(
          attributes.name,
          name,
          'patch request is made for the user with new name',
        );

        const attrs = this.normalizedRequestAttrs('user');

        return users.find(id).update(attrs);
      },
    );

    await page.form.submit.click();
  });
});
```
