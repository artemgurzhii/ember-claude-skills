---
name: ember-4-typescript-early
description: Adopting TypeScript on Ember 4.x — when to use the official setup vs. ember-cli-typescript, the Registry pattern for services and models, why Glint is fragile here and what to skip until 5.x. Use when introducing TS to a 4.x codebase, when planning the typing strategy, or when reviewing a 4.x TS PR.
type: reference
---

# TypeScript on Ember 4.x — The Early Days

4.x is when Ember stopped needing `ember-cli-typescript` as a separate addon. The framework ships its own `.d.ts` files. **But** the template-checking story (Glint) was still maturing during 4.x — most production teams that adopted TS on 4.x **deferred Glint to the 5.x window**.

This skill is the practical setup for "I want TS on 4.x today, with the least risk."

## The decision matrix

| Goal | Recommendation |
|---|---|
| Type-check JavaScript only (services, models, helpers, modifiers, action handlers) | Yes — official 4.x TS support is solid. |
| Type-check templates against component signatures | Maybe. Glint's `ember-loose` environment works on 4.x but is rough; expect regular workarounds. Many teams skip until 5.x. |
| Greenfield 4.x app in 2026 | Don't. Use 5.x and the modern Ember skills. |
| Existing 4.x app, plan to be on 4.x for ≥6 more months | Worth it. Type the JS layer; defer template type-check until the 5.x bump. |
| Existing 4.x app, plan to bump to 5.x soon | Wait. TS infra in 5.x is materially better; do it once. |

## Setup — the JS-only path

If you're not going to fight Glint on 4.x, start here.

### 1. tsconfig

```jsonc
// tsconfig.json
{
  "extends": "@tsconfig/ember/tsconfig.json",
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "node",
    "strict": true,
    "noEmit": true,
    "experimentalDecorators": true,
    "noImplicitAny": true,
    "noUnusedLocals": false,
    "skipLibCheck": true,
    "paths": {
      "my-app/tests/*": ["tests/*"],
      "my-app/*":       ["app/*"],
      "*":              ["types/*"]
    }
  }
}
```

`experimentalDecorators: true` is non-negotiable — `@service`, `@tracked`, `@action`, ember-data decorators all need it.

### 2. Types directory

```
types/
├── global.d.ts
├── ember-data/
│   └── types/
│       └── registries/
│           └── model.d.ts   (you augment this)
└── @ember/
    └── service.d.ts          (you augment Registry here)
```

Create `types/global.d.ts`:

```ts
// types/global.d.ts
import 'ember-source/types';
```

(Some Ember 4.x versions use `'ember/types'`; check your version's `node_modules/ember-source/`.)

### 3. The Registry pattern

The single most important pattern for typed Ember code. **Every service** augments the registry:

```ts
// app/services/cart.ts
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';

export default class CartService extends Service {
  @tracked items: CartItem[] = [];
  add(i: CartItem) { this.items = [...this.items, i]; }
}

declare module '@ember/service' {
  interface Registry {
    cart: CartService;
  }
}
```

After that, `@service declare cart: CartService;` is type-correct app-wide. The `declare` keyword is critical — without it TS emits a class field initializer that runs after the decorator and breaks injection.

**Every ember-data model** does the same:

```ts
// app/models/post.ts
import Model, { attr } from '@ember-data/model';

export default class PostModel extends Model {
  @attr('string') declare title: string;
}

declare module 'ember-data/types/registries/model' {
  export default interface ModelRegistry {
    post: PostModel;
  }
}
```

Now `store.findRecord('post', id)` returns `Promise<PostModel>` without a cast.

### 4. Component signatures

Even if you don't run Glint, **declare component signatures**. They're useful documentation, and they make the eventual Glint adoption nearly free.

```ts
// app/components/post-card.ts
import Component from '@glimmer/component';
import type PostModel from 'my-app/models/post';

export interface PostCardSignature {
  Args: {
    post: PostModel;
    onShare?: (post: PostModel) => void;
  };
  Element: HTMLElement;
  Blocks: {
    default: [];
    actions: [{ share: () => void }];
  };
}

export default class PostCard extends Component<PostCardSignature> {
  // ...
}
```

The signature isn't enforced against the template without Glint, but consumers can pass-through-type-check the args (e.g. when calling the component from another typed component or service).

### 5. CI

Add a single check:

```bash
pnpm exec tsc --noEmit
```

This catches the JS-layer issues. Don't bring in Glint yet on 4.x unless you're prepared for it.

## When to add Glint on 4.x (and when not)

Glint is the type-checker that bridges TS and Handlebars. On 4.x it has two environments:

- `@glint/environment-ember-loose` — for `.ts` + `.hbs` pairs.
- `@glint/environment-ember-template-imports` — for `.gjs`/`.gts` (5.x feature).

On 4.x, only `ember-loose` is relevant. Decide based on team capacity:

**Add Glint if:**
- Your team has senior TS users.
- You can absorb 1–2 weeks of "fixing surprising errors" without slipping features.
- You'll stay on 4.x for ≥1 year.
- You write components with non-trivial args/blocks/element types.

**Skip Glint if:**
- The team is new to TS.
- A 5.x bump is on the roadmap within 6 months.
- The codebase has many leftover untyped helpers/components/modifiers.
- You don't have time for Glint's edge-case workarounds.

If you skip, you still get most of the value: the JS layer is fully typed, services and models are correctly resolved, and the eventual Glint adoption on 5.x is straightforward.

## TypeScript-specific 4.x patterns

### Service injection — always with `declare`

```ts
@service declare router: RouterService;             // ✅
@service router: RouterService = undefined as any;  // ❌ initializer runs after decorator
@service router!: RouterService;                    // ⚠️ hides it but doesn't fix the runtime issue
```

### Decorated fields on classes

```ts
class Cart {
  @tracked declare items: CartItem[];        // initializer-free, decorator runs cleanly
}

// or, with default:
class Cart {
  @tracked items: CartItem[] = [];           // also works; TS emits the init after decorator
}
```

The `declare` form is required when the decorator does something at construction time (like `@service`). For `@tracked`, both forms work; `declare` is slightly safer.

### Template-only components

Without Glint there's no template-type-check, but the signature documents intent:

```ts
// app/components/badge.ts
import templateOnly from '@ember/component/template-only';
import type { TemplateOnlyComponent } from '@ember/component/template-only';

export interface BadgeSignature {
  Args: { label: string };
  Element: HTMLSpanElement;
}

const Badge: TemplateOnlyComponent<BadgeSignature> = templateOnly();
export default Badge;
```

```hbs
{{! app/components/badge.hbs }}
<span class="badge" ...attributes>{{@label}}</span>
```

### Tests in TS

```ts
// tests/integration/components/post-card-test.ts
import { module, test } from 'qunit';
import { setupRenderingTest } from 'my-app/tests/helpers';
import { render } from '@ember/test-helpers';
import { hbs } from 'ember-cli-htmlbars';

interface TestContext {
  post: { title: string };
}

module('Integration | Component | post-card', function (hooks) {
  setupRenderingTest(hooks);

  test('renders the title', async function (this: TestContext, assert) {
    this.post = { title: 'Hello' };
    await render(hbs`<PostCard @post={{this.post}} />`);
    assert.dom('[data-test-title]').hasText('Hello');
  });
});
```

For richer typing, the `@ember/test-helpers` types are correct enough that you don't need to declare context interfaces in most files; they're noted here for completeness.

## Things to skip on 4.x, do later

- **Strict-mode templates / template imports.** A 5.x feature. Don't try to backport.
- **`<template>` blocks.** Same.
- **WarpDrive (next-gen ember-data) typing.** Stick with `@ember-data/model`.
- **Glint `ember-template-imports` environment.** Only relevant in 5.x.
- **Aggressive `noUncheckedIndexedAccess`.** Many ember-data shapes don't satisfy it cleanly on 4.x; turn on after the 5.x bump and the WarpDrive typing rewrite.

## Common 4.x TS mistakes

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot find name '@service'` | Missing `experimentalDecorators` or wrong import. | Import from `@ember/service` and enable the flag. |
| `Property 'router' is used before its initialization` | Forgot `declare`. | `@service declare router: RouterService;`. |
| `store.findRecord('post', '1')` returns `Promise<unknown>` | Missing `ModelRegistry` augmentation. | Add it in the model file. |
| `Type 'X' has no exported member 'Y'` for `ember-data` types | Using post-4.x type paths on 4.x. | Use `@ember-data/model` exports for 4.x. |
| Glint complains about every component's args | Missing or wrong signature. | Add `interface XSignature` and `Component<XSignature>`. |
| Decorated-property initializer runs before decorator | Missing `declare`. | Add `declare` to the property. |

## Verification

- [ ] `tsc --noEmit` passes in CI.
- [ ] `experimentalDecorators: true` set.
- [ ] Every service has a `Registry` augmentation.
- [ ] Every ember-data model has a `ModelRegistry` augmentation.
- [ ] All `@service` properties use `declare`.
- [ ] Component signatures declared (even without Glint).
- [ ] If Glint is on, it's `ember-loose` environment only.

## See also

- `ember-4-octane-only` — what's available in 4.x.
- `ember-4-to-5-migration` — when Glint becomes worth doing.
- Modern reference: [`ember-typescript-and-glint`](../ember-typescript-and-glint).
