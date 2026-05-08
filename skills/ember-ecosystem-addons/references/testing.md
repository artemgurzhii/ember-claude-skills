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

Caveat: release cadence has slowed (last publish 2024-09 as of 2026-05). It still works on modern Ember and is the largest install base, but new projects increasingly reach for [MSW](https://mswjs.io) instead — same ergonomics with cross-framework support and active development. Stay on Mirage if you already have a Mirage server with factories and scenarios; consider MSW for greenfield apps.

## [`ember-a11y-testing`](https://github.com/ember-a11y/ember-a11y-testing)

Wraps axe-core for in-test a11y audits.

```bash
ember install ember-a11y-testing
```

```ts
import a11yAudit from 'ember-a11y-testing/test-support/audit';

await visit('/');
await a11yAudit();
```

## [`@percy/ember`](https://github.com/percy/percy-ember)

Visual regression via Percy. Install only if you have a Percy account.
