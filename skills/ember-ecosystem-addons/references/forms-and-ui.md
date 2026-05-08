# Forms & UI

## [`ember-basic-dropdown`](https://ember-basic-dropdown.com)

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

## [`ember-power-select`](https://ember-power-select.com)

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

For free-text typeahead, use `<PowerSelect>` with `@searchEnabled={{true}}` plus a custom `@search` action — the standalone `ember-power-select-typeahead` companion is unmaintained (last release 2022).

## [`ember-power-calendar`](https://ember-power-calendar.com)

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

## [`ember-power-datepicker`](https://github.com/cibernox/ember-power-datepicker)

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

## [`ember-headless-form`](https://github.com/CrowdStrike/ember-headless-form)

Mainmatter / CrowdStrike-maintained headless form library — handles state, validation wiring, accessibility (label / error / `aria-describedby` plumbing), and submission lifecycle. You provide the markup; the addon manages the form mechanics. Strict-mode `.gts`-friendly and Polaris-aligned.

```bash
pnpm add ember-headless-form
```

```hbs
<HeadlessForm @data={{this.user}} @onSubmit={{this.save}} as |form|>
  <form.Field @name="email" as |field|>
    <field.Label>Email</field.Label>
    <field.Input @type="email" required />
    <field.Errors />
  </form.Field>

  <form.Field @name="password" as |field|>
    <field.Label>Password</field.Label>
    <field.Input @type="password" minlength="8" />
    <field.Errors />
  </form.Field>

  <button type="submit" disabled={{form.isSubmitting}}>Sign in</button>
</HeadlessForm>
```

Pair with a validation library (`yup`, `zod`, `valibot`) via the `@validate` argument when native-constraint validation isn't enough. Reach for this instead of hand-rolling form state in component classes.

## [`ember-changeset`](https://github.com/adopted-ember-addons/ember-changeset)

A buffered, rollback-able wrapper around POJOs and Ember Data records. Tracks edits in memory, exposes `isDirty` / `isInvalid` / `changes` / `errors`, applies the buffer on `save()` / `execute()`, or discards it via `rollback()`. The standard answer to "I want unsaved-form state separate from the canonical model," and the usual companion to `ember-changeset-validations`.

```bash
ember install ember-changeset
```

```ts
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { Changeset } from 'ember-changeset';

export default class UserEditForm extends Component {
  changeset = Changeset(this.args.user);

  @action update(key: string, event: Event) {
    this.changeset.set(key, (event.target as HTMLInputElement).value);
  }

  @action async save() {
    if (this.changeset.isInvalid) return;
    await this.changeset.save();   // applies buffer to the underlying model
  }

  @action cancel() {
    this.changeset.rollback();
  }
}
```

```hbs
<input
  value={{this.changeset.email}}
  {{on "input" (fn this.update "email")}}
/>
{{#if this.changeset.error.email}}
  <p class="error">{{this.changeset.error.email.validation}}</p>
{{/if}}

<button type="button" {{on "click" this.save}} disabled={{this.changeset.isInvalid}}>
  Save
</button>
<button type="button" {{on "click" this.cancel}}>Cancel</button>
```

`save()` calls the underlying model's `save()` if it exists (Ember Data records); otherwise it just applies the buffer. `execute()` applies without saving when you want to commit locally.

## [`ember-changeset-validations`](https://github.com/adopted-ember-addons/ember-changeset-validations)

Drop-in validators for `ember-changeset` — `validatePresence`, `validateLength`, `validateFormat`, `validateNumber`, `validateInclusion`, `validateConfirmation`. Compose into a key-by-key map and pass to the changeset; no hand-rolled validator functions.

```bash
ember install ember-changeset-validations
```

```ts
// app/validations/user.ts
import {
  validatePresence,
  validateLength,
  validateFormat,
  validateNumber,
} from 'ember-changeset-validations/validators';

export default {
  email:    [validatePresence(true), validateFormat({ type: 'email' })],
  password: validateLength({ min: 8, max: 128 }),
  age:      validateNumber({ integer: true, gte: 13 }),
};
```

```ts
import { Changeset } from 'ember-changeset';
import lookupValidator from 'ember-changeset-validations';
import UserValidations from 'my-app/validations/user';

changeset = Changeset(this.args.user, lookupValidator(UserValidations), UserValidations);
```

Notes:
- For new strict-mode (`.gts`) work, prefer `ember-headless-form` + `yup` / `zod` / `valibot` — that path is more actively maintained and aligns with the Polaris form story.
- Reach for `ember-changeset` when editing Ember Data records in place and you want a real `rollback()`, when an existing classic-mode app already uses changesets, or when you need nested-edit support that the headless-form schema validators don't cover cleanly.
- Both addons live under `adopted-ember-addons` — release cadence is slow but stable; they still work on modern Ember.

## [`ember-primitives`](https://github.com/universal-ember/ember-primitives)

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

## [`ember-truth-helpers`](https://github.com/jmurphyau/ember-truth-helpers)

Adds `eq`, `not`, `and`, `or`, `is-equal`, `lt`, `gt` template helpers. Without these, you can't combine booleans in templates.

```bash
ember install ember-truth-helpers
```

Modern Ember (≥ 6.3) ships `eq`, `not`, `and`, `or` as built-in template keywords — no import needed in `.gjs` / `.gts` strict mode. Install `ember-truth-helpers` only when you're on an older Ember version, still authoring loose-mode `.hbs`, or need the extras (`is-equal`, `lt`, `gt`).

## [`@ember/render-modifiers`](https://github.com/emberjs/ember-render-modifiers)

`{{did-insert}}`, `{{did-update}}`, `{{will-destroy}}` modifiers. Escape hatch for "I really need lifecycle here." Often the right answer is a custom modifier (next).

## [`ember-modifier`](https://github.com/ember-modifier/ember-modifier)

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

## [`ember-resources`](https://github.com/NullVoxPopuli/ember-resources)

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

## [`ember-cli-flash`](https://github.com/poteto/ember-cli-flash)

Toast / flash messages with queueing, auto-dismiss, and per-message callbacks. The de-facto notification system in Ember apps — small surface area, exposes a `flashMessages` service.

```bash
ember install ember-cli-flash
```

```ts
// app/components/save-button.ts
import Component from '@glimmer/component';
import { service } from '@ember/service';
import { action } from '@ember/object';
import type FlashMessagesService from 'ember-cli-flash/services/flash-messages';

export default class SaveButton extends Component {
  @service declare flashMessages: FlashMessagesService;

  @action async save() {
    try {
      await this.args.onSave();
      this.flashMessages.success('Saved.', { timeout: 3000 });
    } catch (e) {
      this.flashMessages.danger(`Save failed: ${e.message}`, { sticky: true });
    }
  }
}
```

```hbs
{{! app/templates/application.hbs — render once, anywhere }}
{{#each this.flashMessages.queue as |flash|}}
  <FlashMessage @flash={{flash}} />
{{/each}}
```

The addon ships unstyled — bring your own classes or a Tailwind variant. For richer notification UIs (action buttons, progress, swipe-to-dismiss), pair `ember-primitives` `<Toast>` with manual queue management instead.

## [`ember-shepherd`](https://github.com/shipshapecode/ember-shepherd)

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

## [`ember-page-title`](https://github.com/adopted-ember-addons/ember-page-title)

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

## [`ember-svg-jar`](https://github.com/ivanvotti/ember-svg-jar)

Bundles SVGs and exposes them as `<svg-jar>` components or sprite symbols. Tree-shaken; viewer at `/ember-svg-jar`.

```bash
ember install ember-svg-jar
```

```hbs
{{svg-jar "user-circle" class="w-6 h-6"}}
```
