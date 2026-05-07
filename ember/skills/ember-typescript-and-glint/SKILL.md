---
name: ember-typescript-and-glint
description: TypeScript in Ember — service Registry augmentation, model registry, component signatures, and Glint for type-checked templates (loose mode for .hbs and template-imports mode for .gjs/.gts). Use when adding TS to an Ember app, when fixing service injection types, or when getting "unknown property on this" Glint errors.
type: reference
---

# TypeScript & Glint in Ember

Ember has first-class TypeScript support since Ember 5.1. **Glint** is the type-checker that bridges TS and Handlebars — it's the reason `{{this.user.nmae}}` becomes a compile error.

There are two Glint environments:

| Environment | Files | When to use |
|---|---|---|
| `@glint/environment-ember-loose` | Separate `.hbs` + `.ts`. Templates resolve names via Ember's resolver. | Octane apps with classic colocated templates. |
| `@glint/environment-ember-template-imports` | Single `.gjs` / `.gts` files with `<template>` blocks. Strict-mode resolution. | Polaris-track apps. |

You can run **both at once** during a migration. New components in `.gts`, legacy components in `.ts` + `.hbs`.

## tsconfig essentials

```jsonc
// tsconfig.json
{
  "extends": "@tsconfig/ember/tsconfig.json",
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true,
    "experimentalDecorators": true,
    "paths": {
      "my-app/tests/*": ["tests/*"],
      "my-app/*":       ["app/*"],
      "*":              ["types/*"]
    }
  },
  "glint": {
    "environment": [
      "ember-loose",
      "ember-template-imports"
    ]
  }
}
```

`experimentalDecorators` is required for `@service`, `@tracked`, `@action`, and Ember Data decorators.

## Service Registry — the pattern that ties it all together

For each service, augment `@ember/service` so `@service declare foo: FooService;` is type-correct **app-wide**:

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

Now anywhere:

```ts
import { service } from '@ember/service';
import type CartService from 'my-app/services/cart';

class CheckoutComponent extends Component {
  @service declare cart: CartService;
}
```

`declare` is required — without it, TS emits a class field initializer that runs *after* the decorator and breaks DI.

> Some teams use `@service('shopping-cart') declare cart: CartService;` when the property name differs from the service name. The first arg is the registry key.

## Model Registry — typed `store.findRecord`

```ts
// app/models/post.ts
import Model, { attr, belongsTo } from '@ember-data/model';
import type { AsyncBelongsTo } from '@ember-data/model';
import type UserModel from './user';

export default class PostModel extends Model {
  @attr('string') declare title: string;
  @belongsTo('user', { async: true, inverse: 'posts' })
    declare author: AsyncBelongsTo<UserModel>;
}

declare module 'ember-data/types/registries/model' {
  export default interface ModelRegistry {
    post: PostModel;
  }
}
```

```ts
const post = await this.store.findRecord('post', '42'); // typed as PostModel
```

## Component signatures

A signature describes the component's contract — args in, blocks out, what element gets `...attributes`.

```ts
import Component from '@glimmer/component';

export interface ModalSignature {
  Args: {
    title: string;
    isOpen: boolean;
    onClose: () => void;
  };
  Element: HTMLDialogElement;
  Blocks: {
    default: [];
    actions: [{ close: () => void }];
  };
}

export default class Modal extends Component<ModalSignature> {
  @action close() { this.args.onClose(); }
}
```

In `.hbs` templates, Glint checks `@title` against `Args.title`, `...attributes` against `Element`, and `<:actions as |x|>` against `Blocks.actions`.

For helpers and modifiers, similar signature types exist:

```ts
import { helper } from '@ember/component/helper';

export interface FormatCurrencySignature {
  Args: {
    Positional: [amount: number];
    Named: { currency?: string };
  };
  Return: string;
}

export default helper<FormatCurrencySignature>(([amount], { currency = 'USD' }) =>
  new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount)
);
```

```ts
import { modifier } from 'ember-modifier';

export interface AutoFocusSignature {
  Element: HTMLElement;
  Args: { Positional: []; Named: {} };
}

export default modifier<AutoFocusSignature>((el) => {
  el.focus();
});
```

## Loose-mode template typing — `template-registry.ts`

For separate `.hbs` files, Glint needs to know which name maps to which component/helper/modifier. Convention: each file with a default export augments a global registry.

The simplest pattern with modern `glint-environment-ember-loose`:

```ts
// types/glint-registry.d.ts
import 'ember-source/types';
import 'ember-source/types/preview';
import '@glint/environment-ember-loose';
import 'ember-power-select/glint-registry';
import '@ember/test-helpers/glint-registry';

import type Modal from 'my-app/components/modal';
import type FormatCurrency from 'my-app/helpers/format-currency';

declare module '@glint/environment-ember-loose/registry' {
  export default interface Registry {
    Modal: typeof Modal;
    'format-currency': typeof FormatCurrency;
  }
}
```

Many teams script this with [`glint-loose-mode-codegen`](https://github.com/typed-ember/glint) or via `template-registry-codegen` — for new code, prefer template-imports mode (next), where this file is unnecessary.

## Template-imports mode — `.gjs` / `.gts` (Polaris)

In strict mode, you import the things you use:

```ts
// app/components/post-card.gts
import Component from '@glimmer/component';
import { on } from '@ember/modifier';
import FormatDate from 'my-app/helpers/format-date';
import type PostModel from 'my-app/models/post';

export interface PostCardSignature {
  Args: { post: PostModel };
  Element: HTMLElement;
}

export default class PostCard extends Component<PostCardSignature> {
  <template>
    <article ...attributes>
      <h2>{{@post.title}}</h2>
      <time>{{FormatDate @post.publishedAt format="long"}}</time>
      <button type="button" {{on "click" this.share}}>Share</button>
    </article>
  </template>

  share = () => navigator.share?.({ title: this.args.post.title });
}
```

Type-checking is automatic — no registry augmentation needed. Glint sees the imports.

For template-only components (no class), use `.gjs` with a default-exported template literal:

```ts
// app/components/badge.gts
import type { TOC } from '@ember/component/template-only';

export interface BadgeSignature {
  Args: { label: string };
  Element: HTMLSpanElement;
}

const Badge: TOC<BadgeSignature> = <template>
  <span class="badge" ...attributes>{{@label}}</span>
</template>;

export default Badge;
```

`TOC` = "template-only component". Lightweight, no class instance.

## Running Glint

```bash
pnpm exec glint                      # one-shot type check
pnpm exec glint --watch
pnpm exec tsc --noEmit               # only checks TS, not templates — Glint does both
```

Add `glint` to your CI pipeline. It catches a category of bugs nothing else does.

## Common type errors and fixes

| Error | Cause | Fix |
|---|---|---|
| `Property 'foo' does not exist on type 'this'` in template | Either the field is missing from the class or you forgot `declare`. | Add `@tracked declare foo: T;` or `@service declare foo: FooService;`. |
| `Argument of type 'X' is not assignable to type 'never'` after `@service`, no Registry augmentation | Service has no entry in `Registry`. | Add the `declare module '@ember/service' { interface Registry { ... } }` block. |
| `... is not assignable to parameter of type 'TemplateContext<...>'` in `.hbs` | Component signature mismatched against template. | Update `Args`/`Element`/`Blocks` to match what the template uses. |
| `Type 'string' is not assignable to type 'never'` for arg | The component signature didn't list this arg. | Add it to `Args`. |
| `Cannot find module 'ember-power-select/glint-registry'` | Some addons need an explicit Glint registry import. | Add the import in `types/glint-registry.d.ts`. |

## Codemod path: JS → TS

1. Rename `app/components/foo.js` → `foo.ts`.
2. Add the component signature.
3. Replace `@service session` with `@service declare session: SessionService;`.
4. Add `declare` to every decorated field.
5. Run `pnpm exec glint` and chase errors.

For each addon you import, check if it has a `glint-registry` entry point. If yes, add it to your global registry file.

## Verification

- [ ] `tsconfig.json` has `experimentalDecorators: true` and a `glint.environment` entry.
- [ ] Every service has a `Registry` augmentation.
- [ ] Every model has a `ModelRegistry` augmentation.
- [ ] Every component has a `Signature` (Args + Element + Blocks).
- [ ] `@service` and decorated fields use `declare`.
- [ ] Glint runs in CI as a separate step.
- [ ] (Polaris) New components are `.gts` with imported helpers/modifiers.

## See also

- Official: [typed-ember.com](https://typed-ember.com/) and [glint docs](https://typed-ember.com/docs/glint).
- `ember-octane-fundamentals` — `@service` / `@tracked` mechanics.
- `ember-components-and-templates` — what a signature describes.
- `ember-polaris-migration` — moving to `.gts` / template-imports mode.
