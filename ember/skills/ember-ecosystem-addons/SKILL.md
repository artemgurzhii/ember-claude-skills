---
name: ember-ecosystem-addons
description: Curated reference of top-rated Ember addons (per emberobserver.com) — what each one is for, when to install it, the canonical install command, and a tiny usage example. Covers ember-simple-auth, ember-concurrency, ember-test-selectors, ember-cli-page-object, ember-cli-mirage, ember-basic-dropdown, ember-power-select, ember-power-calendar, ember-power-datepicker, ember-page-title, ember-shepherd, ember-modifier, ember-resources, ember-intl, ember-svg-jar, and more. Use when picking an addon, when debugging "what does this addon do," or when scaffolding a new feature that should reuse community standards.
type: reference
---

# Ember Ecosystem — Top-Rated Addons

Ember's payoff comes from convention *and* from a small set of community addons that every serious app uses. Don't reinvent these.

A short list of the addons on [emberobserver.com](https://emberobserver.com) that should be on the table for almost every app, organized by category.

## Authentication

### [`ember-simple-auth`](https://github.com/mainmatter/ember-simple-auth)

The de-facto auth addon. Handles login, logout, token storage, session restoration, and route gating.

```bash
ember install ember-simple-auth
```

```ts
// app/routes/application.ts — restore session on boot
import Route from '@ember/routing/route';
import { service } from '@ember/service';
import type SessionService from 'ember-simple-auth/services/session';

export default class ApplicationRoute extends Route {
  @service declare session: SessionService;

  async beforeModel() {
    await this.session.setup();
  }
}
```

```ts
// app/routes/dashboard.ts — gate a route
export default class DashboardRoute extends Route {
  @service declare session: SessionService;

  beforeModel(transition) {
    this.session.requireAuthentication(transition, 'login');
  }
}
```

```ts
// app/controllers/login.ts — authenticate
@service declare session: SessionService;

@action async login(event: SubmitEvent) {
  event.preventDefault();
  await this.session.authenticate('authenticator:oauth2', email, password);
}
```

Authenticators (`session-stores/`, `authenticators/`) are pluggable. You'll likely write a small custom authenticator that hits *your* `/login` endpoint.

## Async / orchestration

### [`ember-concurrency`](http://ember-concurrency.com)

Tasks with cancellation, dedupe, and concurrency control. **Install this in any app with non-trivial async UI.**

```bash
ember install ember-concurrency
```

```ts
import { task, restartableTask, dropTask, timeout } from 'ember-concurrency';

export default class SearchBox extends Component {
  searchTask = restartableTask(async (query: string) => {
    await timeout(300);                         // debounce
    if (!query.trim()) return [];
    return this.api.get('/search', { q: query });
  });

  saveTask = dropTask(async (data: FormData) => {
    return this.api.post('/save', data);        // ignore re-clicks while running
  });
}
```

```hbs
<input {{on "input" (perform this.searchTask value="target.value")}} />

{{#if this.searchTask.isRunning}}<Spinner />{{/if}}

{{#each this.searchTask.lastSuccessful.value as |hit|}}
  <SearchHit @hit={{hit}} />
{{/each}}
```

Task modifiers:
- `task` — default; concurrent calls run in parallel.
- `restartableTask` — cancel previous run when a new one starts (search-as-you-type).
- `dropTask` — ignore new calls while one is running (form submit).
- `keepLatestTask` — queue exactly one pending after the current.
- `enqueueTask` — queue all calls, run sequentially.

`ember-concurrency` integrates with test waiters automatically — your tests stay flake-free.

## Testing

### [`ember-test-selectors`](https://github.com/simplabs/ember-test-selectors)

Strips `data-test-*` attributes from production builds. Test the contract without polluting prod HTML.

```bash
ember install ember-test-selectors
```

```hbs
<button data-test-save type="button">Save</button>
```

### [`ember-cli-page-object`](https://ember-cli-page-object.js.org)

Page-object pattern for tests. Encapsulates DOM selectors so test bodies read like behavior, not lookups.

```bash
ember install ember-cli-page-object
```

See `ember-testing` for a full example.

### [`ember-cli-mirage`](https://www.ember-cli-mirage.com)

In-browser fake API server with a database, factories, scenarios, and route handlers. Use for both dev (`mirage/scenarios/default.ts`) and tests.

```bash
ember install ember-cli-mirage
```

### [`ember-a11y-testing`](https://github.com/ember-a11y/ember-a11y-testing)

Wraps axe-core for in-test a11y audits.

```bash
ember install ember-a11y-testing
```

```ts
import a11yAudit from 'ember-a11y-testing/test-support/audit';

await visit('/');
await a11yAudit();
```

### [`@percy/ember`](https://github.com/percy/percy-ember)

Visual regression via Percy. Install only if you have a Percy account.

## Form / UI building blocks

### [`ember-basic-dropdown`](https://ember-basic-dropdown.com)

Headless dropdown primitive — handles positioning, click-outside, focus management, keyboard navigation, and rendering into a wormhole. The foundation `ember-power-select` and `ember-power-datepicker` are built on. Reach for it directly when you need a popover with the trigger / content split and don't want to reinvent positioning.

```bash
ember install ember-basic-dropdown
```

```hbs
<BasicDropdown as |dd|>
  <dd.Trigger class="btn">Filters</dd.Trigger>
  <dd.Content class="dropdown-panel">
    <ul>
      <li><button type="button" {{on "click" (fn dd.actions.close)}}>All</button></li>
      <li><button type="button" {{on "click" (fn dd.actions.close)}}>Mine</button></li>
    </ul>
  </dd.Content>
</BasicDropdown>
```

`dd.actions` exposes `open`, `close`, `toggle`, `reposition` for imperative control. For a fully-styled menu, prefer `ember-primitives` `<Menu>`; reach for `ember-basic-dropdown` when you need raw positioning or are composing a higher-level widget.

### [`ember-power-select`](https://ember-power-select.com)

The de-facto select / combobox / typeahead. Accessible, themeable, async-search–friendly. Built on `ember-basic-dropdown`.

```bash
ember install ember-power-select
```

```hbs
<PowerSelect
  @options={{this.users}}
  @selected={{this.selectedUser}}
  @onChange={{this.setUser}}
  @searchField="name"
  as |user|
>
  {{user.name}}
</PowerSelect>
```

Companion: `ember-power-select-typeahead` for free-text typeahead.

### [`ember-power-calendar`](https://ember-power-calendar.com)

Accessible single-date and date-range calendar. Use standalone for inline calendars or as the picker portion of `ember-power-datepicker`.

```bash
ember install ember-power-calendar
ember install ember-power-calendar-luxon   # or -moment / -date-fns
```

```hbs
<PowerCalendar @selected={{this.date}} @onSelect={{this.setDate}} as |cal|>
  <cal.Nav />
  <cal.Days />
</PowerCalendar>
```

Pick exactly one date-library adapter (`-luxon`, `-moment`, `-date-fns`) — the base addon ships no formatter. For ranges, use `<PowerCalendarRange>` with `@selected={{hash start=... end=...}}`.

### [`ember-power-datepicker`](https://github.com/cibernox/ember-power-datepicker)

Date / range input that wires `ember-power-calendar` (the calendar) into `ember-basic-dropdown` (the popover). Drop-in picker for forms.

```bash
ember install ember-power-datepicker
```

```hbs
<PowerDatepicker @selected={{this.startDate}} @onSelect={{this.setStart}} as |dp|>
  <dp.Trigger class="input">
    {{or (format-date this.startDate) "Pick a date"}}
  </dp.Trigger>
  <dp.Content>
    <dp.Nav />
    <dp.Days />
  </dp.Content>
</PowerDatepicker>
```

Same date-library adapter requirement as `ember-power-calendar` — install one of `-luxon` / `-moment` / `-date-fns`.

### [`ember-primitives`](https://github.com/universal-ember/ember-primitives)

Headless, accessible UI primitives for Ember — the rough equivalent of Radix UI / shadcn for the React world. Maintained by NullVoxPopuli. Ships unstyled `<Menu>`, `<Dialog>`, `<Popover>`, `<Switch>`, `<Tabs>`, `<Tooltip>`, `<Form>`, `<DropdownMenu>`, and a growing list of patterns. `.gts`-first, Polaris-friendly, Glint-typed.

Reach for `ember-primitives` when you want **behavior + a11y + focus management for free** but you want to own the styling. Pair it with Tailwind or your design tokens.

```bash
pnpm add ember-primitives
```

```ts
// app/components/share-menu.gts
import Component from '@glimmer/component';
import { Menu } from 'ember-primitives';
import { on } from '@ember/modifier';

export default class ShareMenu extends Component {
  copyLink = () => navigator.clipboard.writeText(window.location.href);

  <template>
    <Menu as |m|>
      <m.Trigger class="btn">Share</m.Trigger>
      <m.Content class="menu">
        <m.Item {{on "click" this.copyLink}}>Copy link</m.Item>
        <m.Item>Email…</m.Item>
        <m.Separator />
        <m.Item disabled>Print</m.Item>
      </m.Content>
    </Menu>
  </template>
}
```

Notes:
- Each primitive yields a hash of contextual sub-components — same shape as the contextual-component pattern documented in `ember-components-and-templates`.
- Strict mode (`.gts`) is the canonical authoring environment. The classic loose-mode resolver path works but is not the focus.
- Compose with `ember-modifier` and `ember-resources` for custom interactions; `ember-primitives` rarely needs to be subclassed.
- For "I want a complete styled component library, not primitives," look elsewhere — primitives are intentionally unstyled. There is no canonical "shadcn for Ember" yet; pairing `ember-primitives` with a small set of project-specific styled wrappers is the current best path.

### [`ember-truth-helpers`](https://github.com/jmurphyau/ember-truth-helpers) (or `@ember/truth-helpers` in 5+)

Adds `eq`, `not`, `and`, `or`, `is-equal`, `lt`, `gt` template helpers. Without these, you can't combine booleans in templates.

```bash
ember install ember-truth-helpers
```

### [`@ember/render-modifiers`](https://github.com/emberjs/ember-render-modifiers)

`{{did-insert}}`, `{{did-update}}`, `{{will-destroy}}` modifiers. Escape hatch for "I really need lifecycle here." Often the right answer is a custom modifier (next).

### [`ember-modifier`](https://github.com/ember-modifier/ember-modifier)

Author your own modifiers — element-attached behavior with cleanup.

```bash
ember install ember-modifier
```

```ts
import { modifier } from 'ember-modifier';

export default modifier((el: HTMLElement) => {
  el.focus();
});
```

### [`ember-resources`](https://github.com/NullVoxPopuli/ember-resources)

A "resource" is a class-based abstraction for *time-varying values* — fetches, websockets, intervals, timers — with proper teardown and reactivity.

```bash
ember install ember-resources
```

```ts
import { resource } from 'ember-resources';
import { tracked } from '@glimmer/tracking';

class Clock {
  @tracked now = new Date();
}

const clockResource = resource(({ on }) => {
  const state = new Clock();
  const id = setInterval(() => state.now = new Date(), 1000);
  on.cleanup(() => clearInterval(id));
  return state;
});
```

Use resources for anything that produces values *over time*. Combine with `ember-concurrency` (one-shot async) for a powerful split.

## Internationalization & formatting

### [`ember-intl`](https://github.com/ember-intl/ember-intl)

ICU message formatting, locale loading, pluralization, date/number formatting. Wraps the browser's `Intl.*` API.

```bash
ember install ember-intl
```

```hbs
{{t "post.commentCount" count=@post.comments.length}}
{{format-date @post.publishedAt format="long"}}
{{format-number @order.total style="currency" currency="USD"}}
```

```yaml
# translations/en-us.yaml
post:
  commentCount: "{count, plural, =0 {no comments} one {one comment} other {# comments}}"
```

## Assets

### [`ember-svg-jar`](https://github.com/ivanvotti/ember-svg-jar)

Bundles SVGs and exposes them as `<svg-jar>` components or sprite symbols. Tree-shaken; viewer at `/ember-svg-jar`.

```bash
ember install ember-svg-jar
```

```hbs
{{svg-jar "user-circle" class="w-6 h-6"}}
```

## Onboarding / guided tours

### [`ember-shepherd`](https://github.com/shipshapecode/ember-shepherd)

Ember wrapper around [Shepherd.js](https://shepherdjs.dev) — guided tours and onboarding tooltips. Exposes a `tour` service for adding steps, starting / cancelling, and listening to lifecycle events. Use it for first-run product tours, feature spotlights, and contextual help.

```bash
ember install ember-shepherd
```

```ts
// app/components/onboarding-trigger.ts
import Component from '@glimmer/component';
import { service } from '@ember/service';
import { action } from '@ember/object';
import type TourService from 'ember-shepherd/services/tour';

const STEPS = [
  {
    id: 'welcome',
    title: 'Welcome',
    text: 'Let us show you around.',
    attachTo: { element: '.app-header', on: 'bottom' },
    buttons: [{ text: 'Next', action() { return this.next(); } }],
  },
  {
    id: 'compose',
    title: 'Compose',
    text: 'Click here to start a draft.',
    attachTo: { element: '[data-test-compose]', on: 'right' },
    buttons: [
      { text: 'Back', action() { return this.back(); } },
      { text: 'Done', action() { return this.complete(); } },
    ],
  },
];

export default class OnboardingTrigger extends Component {
  @service declare tour: TourService;

  @action async start() {
    this.tour.set('defaultStepOptions', { cancelIcon: { enabled: true } });
    this.tour.addSteps(STEPS);
    await this.tour.start();
  }
}
```

Notes:
- Import the bundled CSS (`shepherd.js/dist/css/shepherd.css`) or theme manually — the addon ships behavior, not styles.
- Each step's `buttons[].action` runs in the tour-step context — use `this.next()`, `this.back()`, `this.complete()`, `this.cancel()`.
- For step targets that mount asynchronously, wait for them before calling `start()` — otherwise the popper attaches to nothing and the step appears centered.
- Listen for `complete` / `cancel` on `this.tour` to persist "user has seen onboarding" so it doesn't replay on every visit.

## Document head / metadata

### [`ember-page-title`](https://github.com/adopted-ember-addons/ember-page-title)

Set `<title>` declaratively from any template. Multiple `{{page-title}}` calls across nested routes are joined automatically — child routes prepend, parents trail.

```bash
ember install ember-page-title
```

```hbs
{{! app/templates/application.hbs }}
{{page-title "Acme"}}

{{! app/templates/posts.hbs }}
{{page-title "Posts"}}

{{! app/templates/posts/show.hbs }}
{{page-title @model.title separator=" — "}}
```

Renders `<title>My First Post — Posts — Acme</title>`. Use the `separator` argument once at the leaf — it controls how that segment joins to its parent. Prefer this to manually setting `document.title` from routes; `ember-page-title` updates the head reactively as routes transition, plays well with FastBoot, and is the convention every Polaris app expects.

## Linting / quality

### [`ember-template-lint`](https://github.com/ember-template-lint/ember-template-lint)

The linter for `.hbs` and `.gjs` templates. Comes with sensible defaults (`recommended` and `octane` configs). Catches `<input @value=...>` mistakes, missing `type=` on buttons, accessibility issues (`a11y-accessibility` rules), and Octane anti-patterns.

```bash
pnpm add -D ember-template-lint
pnpm exec ember-template-lint .
```

### [`@embroider/*`](https://github.com/embroider-build/embroider)

Embroider is the modern build pipeline for Ember — it produces standard ES modules, supports tree-shaking, and is the foundation for Vite-based dev servers (`ember-cli-vite`). Most modern apps run Embroider in "Optimized" mode.

```bash
ember install @embroider/core @embroider/compat @embroider/webpack
```

Polaris assumes Embroider.

## Utility addons worth knowing

| Addon | What it gives you |
|---|---|
| [`tracked-built-ins`](https://github.com/tracked-tools/tracked-built-ins) | `TrackedArray`, `TrackedMap`, `TrackedSet`, `TrackedWeakMap` — reactive primitives with a familiar API. |
| [`ember-element-helper`](https://github.com/tildeio/ember-element-helper) | `(element "h1")` to render dynamic tags. |
| [`ember-on-helper`](https://github.com/buschtoens/ember-on-helper) | Functional `(on "click" this.fn)` for use in `(component ...)` and contextual components. |
| [`ember-keyboard`](https://github.com/adopted-ember-addons/ember-keyboard) | Declarative keyboard shortcuts with priority/scope. |
| [`ember-animated`](http://ember-animated.com) | Animations and transitions tied to Ember's render cycle. |
| [`ember-cli-deprecation-workflow`](https://github.com/mixonic/ember-cli-deprecation-workflow) | Bulk-snooze deprecations during upgrades, fix them in waves. |

## Dev tooling

| Tool | Purpose |
|---|---|
| [Ember Inspector](https://github.com/emberjs/ember-inspector) (browser ext) | Routes, components, services, store records — your X-ray. |
| [`ember-auto-import`](https://github.com/embroider-build/ember-auto-import) | `import 'lodash'` from npm without an addon wrapper. (Default in modern setups.) |

## Choosing well — heuristics

1. **Check emberobserver.com**: install count, score, last release, TypeScript support. Sub-1k installs and stale releases are red flags.
2. **Prefer Mainmatter / NullVoxPopuli / ember-modifier / ember-intl–maintained addons** — these orgs ship reliably.
3. **Beware "Ember 1.x rewrites"** — anything written before Octane often relies on classic patterns.
4. **Read the README + look at the test suite** before installing. Quality of tests = quality of addon.
5. **Don't pile on convenience addons** — every dep is upgrade weight. If the addon wraps 30 lines, copy the 30 lines.

## When NOT to install an addon

- For one-off DOM glue, write a custom modifier (3 lines).
- For one-off transformations, write a helper module.
- For state with no async, just write a class.
- For private API wrappers around `getOwner`, just call `getOwner` once.

## Verification when adopting a new addon

- [ ] Score and install count on emberobserver.com are healthy.
- [ ] Last release is within ~12 months (or there's a maintained fork).
- [ ] The addon has TypeScript types or Glint signatures.
- [ ] You've read the README; you know what one feature you need.
- [ ] You've added it to `package.json` (not duplicated by a transitive dep).

## See also

- `ember-octane-fundamentals` — the patterns these addons assume.
- `ember-testing` — Mirage / page-object / test-selectors integration.
- `ember-services-and-state` — `ember-concurrency` orchestration patterns.
- `ember-typescript-and-glint` — typing addons that ship Glint signatures.
