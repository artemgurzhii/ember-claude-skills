---
name: ember-2-classic-patterns
description: The canonical Ember 2.x mental model — Ember.Object.extend, Ember.computed properties, classic Ember.Component, the actions hash and {{action}}, mixins, observers, run loop scheduling, and two-way data binding via {{mut}}. Use when reading or maintaining 2.x code, when interpreting unfamiliar deprecation messages, or when explaining why a 2.x pattern translates the way it does in modern Ember.
type: reference
---

# Ember 2.x Classic Patterns

This skill describes how Ember code was written between Aug 2015 (2.0) and Dec 2017 (2.18 LTS). Almost every pattern below is **deprecated or removed** in modern Ember, but you'll see it in any 2.x repo that hasn't been modernized.

If you're trying to *understand* legacy code, this is the right reference. If you're trying to write *new* code, use the modern Ember skills instead (`ember-octane-fundamentals` and friends).

## The base class — `Ember.Object`

In 2.x, almost everything is built on `Ember.Object`:

```js
// app/components/user-card.js
import Ember from 'ember';

export default Ember.Component.extend({
  classNames: ['user-card'],
  user: null,

  fullName: Ember.computed('user.firstName', 'user.lastName', function () {
    const first = this.get('user.firstName');
    const last  = this.get('user.lastName');
    return `${first} ${last}`;
  }),

  actions: {
    save() {
      const user = this.get('user');
      user.save();
    }
  }
});
```

Things to notice:

- **`Ember.Component.extend({...})`** — not a class. Defining `class extends Component { ... }` does *not* work in 2.x.
- **`this.get(...)` / `this.set(...)`** — direct property access via `this.foo` works for plain values but not for computed deps; `get`/`set` is the canonical accessor.
- **`actions: {...}`** — a special hash. Templates invoke them via `{{action "save"}}`, which bubbles up the route hierarchy unless intercepted.
- **`classNames: [...]`** — an `Ember.Component` field that becomes class attributes on the wrapping `<div>`.

### Mixins

Composition was achieved with mixins, not interfaces:

```js
// app/mixins/audited.js
import Ember from 'ember';

export default Ember.Mixin.create({
  audit: Ember.on('init', function () {
    Ember.Logger.log('init', this);
  })
});
```

```js
// app/components/foo.js
import Ember from 'ember';
import Audited from 'my-app/mixins/audited';

export default Ember.Component.extend(Audited, {
  // ...
});
```

Mixins were popular for "this thing also does X." They became a maintenance liability — multiple mixins with overlapping property names produce hard-to-debug merges. **Modern Ember has removed mixins**; refactor them into services or higher-order components when migrating.

### Observers and `Ember.on`

```js
nameDidChange: Ember.observer('user.name', function () {
  this.notifyTheServer();
}),

setupOnInit: Ember.on('init', function () {
  this.set('cache', new Map());
})
```

Observers and `on('init', ...)` are **deprecated paths** in modern Ember. They cause subtle ordering bugs (an observer can fire mid-construction). Migrate observers to:

- A computed property + `@cached` (when you can compute the answer instead of reacting).
- An `@action` that the caller invokes.
- A modifier (when the work touches the DOM).

## Computed properties

```js
fullName: Ember.computed('firstName', 'lastName', function () {
  return `${this.get('firstName')} ${this.get('lastName')}`;
}),

isFull: Ember.computed.equal('items.length', 10),
total: Ember.computed.sum('items.@each.price'),
```

The `'items.@each.price'` syntax tells Ember to recompute when *any item's `price` field* changes. It's powerful, but its dependency model is what `@tracked` replaced — modern Ember reads dependencies automatically; 2.x requires you to declare them as strings.

Common computed macros you'll see in 2.x code:

| Macro | Effect |
|---|---|
| `Ember.computed.alias('foo')` | Two-way alias to another property. |
| `Ember.computed.reads('foo')` | One-way read of another property. |
| `Ember.computed.equal('x', value)` | Boolean: is `x === value`. |
| `Ember.computed.gt('items.length', 0)` | Boolean: is greater than. |
| `Ember.computed.sum('numbers')` | Sum of an array. |
| `Ember.computed.mapBy('users', 'name')` | Project a key out of each item. |
| `Ember.computed.filterBy('items', 'isDone')` | Filter array by truthy property. |
| `Ember.computed.or('a', 'b')` / `.and(...)` | Logical macros. |

## Classic component anatomy

```js
import Ember from 'ember';

export default Ember.Component.extend({
  // Customizes the wrapper element:
  tagName: 'article',
  classNames: ['post'],
  classNameBindings: ['isFeatured:post--featured'],
  attributeBindings: ['data-test-post:dataTestPost'],

  // Lifecycle (these still exist in 2.x):
  didInsertElement() {
    this._super(...arguments);
    this.$('.first-input').focus();   // jQuery
  },
  willDestroyElement() {
    this._super(...arguments);
    this.tearDownPlugin();
  },

  actions: {
    publish() {
      this.set('isPublished', true);
      this.sendAction('onPublish', this.get('post'));   // see below
    }
  }
});
```

Notes:

- **`tagName`, `classNames`, `classNameBindings`, `attributeBindings`** — these auto-render a wrapping element. In Octane, `@glimmer/component` has no wrapper element; you express the same intent with `...attributes` on whichever root element you choose.
- **`this.$(...)`** — jQuery is a hard dependency in 2.x. Use it sparingly even within 2.x: jQuery becomes optional in 3.x and is removed in 4.x.
- **`this._super(...arguments)`** — required in any overridden hook. Forgetting it is a common 2.x bug.
- **`sendAction`** — fires a string-named action up the parent chain. Replaced by closure actions in late-2.x and by passing callbacks in Octane.

## Templates — HTMLBars

```hbs
{{!-- Curly invocation (the 2.x default) --}}
{{user-card user=this.user onSave=(action "save")}}

{{!-- Two-way binding via mut --}}
{{input value=(mut name) placeholder="Your name"}}

{{!-- Element-space action --}}
<button {{action "save" post}}>Save</button>

{{!-- Iteration --}}
{{#each posts key="id" as |post|}}
  {{post-card post=post}}
{{/each}}
```

Things that look strange to a modern eye:

- **Curly invocation:** `{{user-card ...}}` is the 2.x default. Angle-bracket invocation (`<UserCard ...>`) was added in 3.4 and became canonical in Octane.
- **`(mut foo)`:** Creates a writable cell. Used heavily for two-way bindings on inputs and for "let the child mutate the parent's state." The pattern is gone in Octane.
- **`{{action ...}}`:** Three uses — passed as a value (`onSave=(action "save")`), attached to an element (`{{action "save"}}`), or fired from JS (`this.sendAction(...)`). All three are replaced by closure functions and `{{on}}` in Octane.
- **`key="id"`** in `{{#each}}`: keeps row identity stable. Still works in modern Ember.

## Routes — the part that aged well

Routing is the most stable part of Ember across versions. The route hooks (`beforeModel`, `model`, `afterModel`, `redirect`, `setupController`) work the same way. The biggest 2.x-only pieces:

```js
// app/routes/posts/show.js
import Ember from 'ember';

export default Ember.Route.extend({
  model(params) {
    return this.store.findRecord('post', params.post_id);
  },
  actions: {
    willTransition(transition) {
      if (this.get('controller.hasUnsaved') && !confirm('Discard?')) {
        transition.abort();
      }
    }
  }
});
```

`transitionTo` is on the route itself in 2.x: `this.transitionTo('login')`. That moved to the `RouterService` later.

## Controllers — heavier than they should have been

In 2.x, controllers were the natural home for actions, computed properties, and template-bound state. Stale state across navigations was a frequent bug.

```js
// app/controllers/posts/index.js
import Ember from 'ember';

export default Ember.Controller.extend({
  queryParams: ['sort', 'page'],
  sort: 'recent',
  page: 1,

  filteredPosts: Ember.computed.filterBy('model', 'isPublished'),

  actions: {
    nextPage() { this.incrementProperty('page'); }
  }
});
```

By Octane, the recommendation became: **controllers exist only for query params**, everything else moves to services or components. When migrating, audit each controller and most of the code can move out.

## Ember Data (in the 2.x era)

```js
// app/models/post.js
import DS from 'ember-data';

export default DS.Model.extend({
  title: DS.attr('string'),
  body:  DS.attr('string'),
  author: DS.belongsTo('user'),
  comments: DS.hasMany('comment')
});
```

`DS.Model.extend(...)`, `DS.attr`, `DS.belongsTo`, `DS.hasMany`. Modern Ember Data uses native classes and decorators (`import Model, { attr, belongsTo } from '@ember-data/model'`). Inverses are *implicit* in 2.x — they became required-explicit in later versions, and missing inverses are one of the most common bugs encountered when upgrading.

## The run loop

The Ember run loop schedules work into queues (`actions`, `routerTransitions`, `render`, `afterRender`, `destroy`). In 2.x you'll see:

```js
import Ember from 'ember';

Ember.run.scheduleOnce('afterRender', this, 'measureWidth');
Ember.run.later(this, 'pollServer', 5000);
Ember.run.cancel(this._timer);
```

Modern Ember exposes the same primitives via `@ember/runloop`:

```js
import { scheduleOnce, later, cancel } from '@ember/runloop';
```

When migrating, this is a near-mechanical rename.

## jQuery

Ember 2.x ships with jQuery as a hard peer dep. It's exposed as `Ember.$` and as `this.$()` on components. Common 2.x usages:

- DOM manipulation in `didInsertElement`.
- AJAX (`Ember.$.ajax`).
- Plugin integration (any `$(selector).whatever()` library).

In modern Ember, jQuery is **removed**. Migrating means replacing `this.$()` with native DOM APIs, `Ember.$.ajax` with `fetch`, and any plugin integration with a custom modifier.

## Common 2.x mistakes to recognize while reading code

| Symptom | What's actually happening |
|---|---|
| `this.foo` returns `undefined` for a computed property | You need `this.get('foo')` for computed and proxy properties. |
| Computed property never recomputes | A dependency string is wrong (typo, missing `@each`). |
| Action fires but nothing happens | The action wasn't found in the action chain — it bubbled to the route, didn't match, and was silently dropped. |
| Two components seem to share state mysteriously | An `alias` macro or a mixin closure created a shared reference. |
| `_super` not called → "weirdness" | A required hook chain wasn't continued. Always call `this._super(...arguments)`. |
| `Cannot read property 'set' of undefined` after a route transition | Code ran on a destroyed component. Guard with `if (this.isDestroyed || this.isDestroying) return;`. |

## Verification when reading 2.x code

- [ ] You recognize `Ember.Object.extend({...})` as the class definition pattern.
- [ ] You read `this.get('a.b.c')` as nested-key access through the property system.
- [ ] You can spot `(mut foo)` and `{{action "x"}}` and explain what they do.
- [ ] You know `Ember.computed('a.b', 'c.@each.x', function() {...})` is dependency-keyed.
- [ ] You can identify `this.$(...)` as jQuery.
- [ ] You know mixins and observers are migration-time refactor candidates.

## See also

- `ember-2-testing` — the test API of this era.
- `ember-2-recommendations` — surviving on 2.x.
- `ember-2-to-3-migration` — the upgrade path.
- Official: [emberjs.com/api archive](https://emberjs.com/api/) (browse to "2.18").
