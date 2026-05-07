---
name: ember-data
description: Ember Data — store, models, attributes, relationships, adapters, serializers, JSON:API conventions, the @ember-data/request request manager, and when to drop down to plain fetch. Use when modeling server data, debugging unexpected null relationships, choosing between findRecord/query/peekRecord, or migrating to the new request system.
type: reference
---

# Ember Data

Ember Data is a normalized, identity-mapped store for your server data. Two ways to think about it:

- **Old school (legacy mental model):** A model layer with conventional Adapter/Serializer pipelines that target JSON:API by default. Suited to REST APIs.
- **New school (post-RFC 716):** A request-builder + request-manager system (`@ember-data/request`) with reactive cache. Adapters/Serializers become optional; you build requests directly.

Modern apps mix both. New code should prefer `@ember-data/request` for fetches and the cache for reactivity, but keep `Model` classes and `findRecord`/`query` for the convenience.

## Models

```ts
// app/models/post.ts
import Model, { attr, belongsTo, hasMany } from '@ember-data/model';
import type { AsyncBelongsTo, AsyncHasMany } from '@ember-data/model';
import type UserModel from './user';
import type CommentModel from './comment';

export default class PostModel extends Model {
  @attr('string') declare title: string;
  @attr('string') declare body: string;
  @attr('date')   declare publishedAt: Date | null;
  @attr('boolean', { defaultValue: false }) declare isDraft: boolean;

  @belongsTo('user', { async: true, inverse: 'posts' })
    declare author: AsyncBelongsTo<UserModel>;

  @hasMany('comment', { async: true, inverse: 'post' })
    declare comments: AsyncHasMany<CommentModel>;

  get isPublished(): boolean {
    return !this.isDraft && this.publishedAt !== null;
  }
}

declare module 'ember-data/types/registries/model' {
  export default interface ModelRegistry {
    post: PostModel;
  }
}
```

- `@attr(transformName, options)` declares a transformed primitive. Built-in transforms: `string`, `number`, `boolean`, `date`. Custom transforms in `app/transforms/`.
- `@belongsTo` / `@hasMany` declare relationships. **Always specify `async` and `inverse`** explicitly — implicit inverses are deprecated.
- The `ModelRegistry` augmentation gives `store.findRecord('post', id)` a typed return.

## The store

`@service('store')` is the entry point. Common APIs:

```ts
@service declare store: Store;

// Fetch by id (cache-first, falls back to network)
const post = await this.store.findRecord('post', id);
const fresh = await this.store.findRecord('post', id, { reload: true });
const background = await this.store.findRecord('post', id, { backgroundReload: true });

// Query the server
const recent = await this.store.query('post', { sort: 'recent', page: 1 });

// All currently loaded records (no network)
const cached = this.store.peekAll('post');
const one = this.store.peekRecord('post', id);

// Create / save / unload
const draft = this.store.createRecord('post', { title: 'New' });
await draft.save();          // POST
draft.title = 'Updated';
await draft.save();          // PATCH

await draft.destroyRecord(); // DELETE + remove from store
draft.unloadRecord();        // remove from store, no network
```

### `findRecord` vs `query` vs `queryRecord` vs `peek*`

| Method | Returns | Hits network? |
|---|---|---|
| `findRecord(type, id)` | One record | Only if not in cache or `reload: true`. |
| `findAll(type)` | All of type | Yes (background-reloads if cached). |
| `query(type, params)` | RecordArray | Always. |
| `queryRecord(type, params)` | One record (or `null`) | Always. |
| `peekRecord(type, id)` | One or `null` | Never. |
| `peekAll(type)` | Live array of cached records | Never. |

Don't use `findAll` casually — for a list endpoint with filters/pagination, use `query`.

## Relationships

```ts
// Async — returns a Promise<Record>
const author = await post.author;
const comments = await post.comments;

// Sync (when async: false) — returns directly
post.tags.forEach(t => /* ... */);

// In templates, async relationships unwrap automatically:
// {{post.author.name}}
```

Mutating relationships:

```ts
// belongsTo
post.author = newUser;        // assignment
await post.save();

// hasMany — operate on the resolved array
const comments = await post.comments;
comments.push(newComment);    // tracked via the relationship
// or: comments.splice(0, 0, newComment);
await post.save();
```

> Rule: **inverse must be set explicitly.** `@hasMany('comment', { async: true, inverse: 'post' })` and on `comment`: `@belongsTo('post', { async: false, inverse: 'comments' })`. Without inverses, deletes won't propagate, additions can duplicate, and you'll spend hours debugging "why is the array empty."

## Adapter + Serializer (legacy pipeline)

By default, requests go through `app/adapters/application.ts`. The default adapter speaks JSON:API.

```ts
// app/adapters/application.ts
import JSONAPIAdapter from '@ember-data/adapter/json-api';
import { service } from '@ember/service';

export default class ApplicationAdapter extends JSONAPIAdapter {
  @service declare session: SessionService;
  host = 'https://api.example.com';
  namespace = 'v1';

  get headers() {
    return {
      Authorization: `Bearer ${this.session.token}`,
      Accept: 'application/vnd.api+json',
    };
  }
}
```

Common adapter overrides:
- `host`, `namespace`, `headers`.
- `pathForType(type)` for non-pluralized URLs (e.g. `'fish'` → `'fish'` instead of `'fishes'`).
- `urlForFindRecord(id, type, snapshot)` for irregular endpoints.

Serializers translate API payloads to JSON:API shape:

```ts
// app/serializers/application.ts
import RESTSerializer from '@ember-data/serializer/rest';
// or:  JSONSerializer, JSONAPISerializer
```

If your API is **already JSON:API**, you usually don't need a serializer. If it's REST or bespoke, you'll write `normalizeResponse` overrides — at which point ask yourself whether `@ember-data/request` (below) is a better fit for new endpoints.

## `@ember-data/request` — the new request layer

For new code, prefer building a **request manager** with handlers:

```ts
// app/services/request-manager.ts
import RequestManager from '@ember-data/request';
import Fetch from '@ember-data/request/fetch';
import { CacheHandler } from '@ember-data/store';

const manager = new RequestManager();
manager.use([Fetch]);
manager.useCache(CacheHandler);

export default manager;
```

Then make requests via builders:

```ts
import { findRecord } from '@ember-data/json-api/request';

const { content: post } = await this.store.request(findRecord('post', '42'));
```

For custom REST endpoints, write a builder:

```ts
function activatePost(id: string) {
  return {
    url: `/api/posts/${id}/activate`,
    method: 'POST',
    cacheOptions: { reload: true },
  };
}

await this.store.request(activatePost(post.id));
```

Builders are plain functions — easy to test, easy to type. You no longer need a custom adapter for one-off endpoints.

## Saving and validation

```ts
const post = this.store.createRecord('post', { title: 'Draft' });

post.title = '';                  // local mutation
const valid = post.title.length > 0;

try {
  await post.save();
} catch (err) {
  // err is a JSON:API errors document by default
  console.log(post.errors);       // accessor for per-attribute errors
  console.log(post.isValid);      // false
}
```

Server-returned errors land on `record.errors`, which is reactive — bind it to your form UI.

## Reloading and background reloads

```ts
await post.reload();              // explicit refetch

await post.belongsTo('author').reload();
await post.hasMany('comments').reload();
```

In a route's `model`:

```ts
async model({ id }: { id: string }) {
  return this.store.findRecord('post', id, {
    backgroundReload: true,       // returns cached, refreshes silently
  });
}
```

## Performance considerations

- **The store is identity-mapped:** loading the same record twice from different endpoints merges into one record. Stale data on one endpoint can leak into another.
- **`includes` are your friend:** `this.store.findRecord('post', id, { include: 'author,comments.author' })` issues one request and hydrates relationships. Avoids N+1.
- **`reload: true` invalidates cache for that resource.** Don't pass it casually.
- **`peek*` is synchronous and cheap.** For derived UI, prefer peek + tracked recompute over re-fetching.
- **Avoid `findAll` for lists.** It often blows up bandwidth on production data sets.

## When NOT to use Ember Data

- One-off RPC endpoints (search, analytics events, "ping"). Use plain `fetch` or `@ember-data/request` without a model.
- Streaming or push-only data (websockets, SSE) — push into the store via `store.push({ data: { ... } })` if you want it cached, or keep it in a service.
- Massively denormalized aggregate endpoints (dashboards). These rarely fit the model abstraction; prefer raw fetch + tracked POJOs.

## TS gotchas

- **Always declare async/sync types:** use `AsyncBelongsTo<T>` / `AsyncHasMany<T>` for `async: true`, plain `T` / `T[]` for sync. Glint reads these for template type-checks.
- **`declare` keyword** on attribute fields prevents TS from emitting an initializer that breaks the decorator.
- **`ModelRegistry` augmentation** gives `store.findRecord('post', id)` a typed `Promise<PostModel>`.

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Missing `inverse:` | Inconsistent relationship state, deprecation warnings. | Add `inverse: 'fieldName'` on both sides. |
| Mutating `post.tags` in place via array methods | Sometimes works, sometimes not. | Resolve the relationship, then use array methods on the resolved array (`await post.tags`). |
| Using `findAll` everywhere | Fetches the whole table. | Use `query` with filters/pagination. |
| Custom REST API + custom serializer + custom adapter | Brittle, lots of code to debug. | Build with `@ember-data/request` builders. |
| `await store.findRecord(...)` from a component without route preload | Loading flicker, no error boundary. | Fetch in the route's `model` hook. |
| Forgetting `Accept: application/vnd.api+json` for JSON:API | Server returns wrong content type. | Set in adapter `headers`. |

## Verification

- [ ] Every relationship has explicit `async` and `inverse`.
- [ ] Each model has a `ModelRegistry` augmentation.
- [ ] List endpoints use `query`, not `findAll`.
- [ ] Route-level data is fetched in `model`, not in component constructors.
- [ ] One-off endpoints use `@ember-data/request` builders or `fetch`, not custom adapters.
- [ ] No silent overrides of `normalizeResponse` without tests.

## See also

- Official deep dive: [api.emberjs.com](https://api.emberjs.com) and the [WarpDrive guides](https://api.emberjs.com/ember-data/) (newer Ember Data docs are publishing under the WarpDrive brand).
- `ember-routing-and-models` — fetching in routes.
- `ember-ecosystem-addons` — `ember-concurrency` for cancellable searches that hit the store.
- `ember-testing` — Mirage scenarios + `setupMirage` for store-driven tests.
