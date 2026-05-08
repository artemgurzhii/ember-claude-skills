# Observability — error monitoring

## [`@sentry/ember`](https://docs.sentry.io/platforms/javascript/guides/ember/)

The official Sentry SDK for Ember. Captures unhandled errors from router transitions, actions, ember-concurrency tasks, and template rendering, plus performance traces for route transitions, components, and ember-data requests. Hooks into Ember's `instrument` events so coverage is broader than a plain `@sentry/browser` install.

```bash
ember install @sentry/ember
```

The SDK is heavy (tracing + replay together push ~100 KB gzipped). For initial-load performance, keep `@sentry/ember` out of the main bundle by lazy-loading it from an initializer that defers app readiness until the chunk lands. Pair that with a dedicated `app/sentry.ts` file that owns the `Sentry.init` config — `app/app.ts` stays small and `Sentry.init` lives in one obvious place.

```ts
// app/sentry.ts — owns the Sentry.init config; runs after window.Sentry is set
import config from 'my-app/config/environment';

declare global {
  interface Window {
    Sentry: typeof import('@sentry/ember');
  }
}

window.Sentry.init({
  dsn: config.sentry.dsn,
  environment: config.environment,
  release: config.APP.version,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    window.Sentry.browserTracingIntegration(),
    window.Sentry.replayIntegration({ maskAllText: false, blockAllMedia: true }),
  ],
});
```

```ts
// app/initializers/load-externals.ts — lazy-load Sentry as its own chunk
import Application from '@ember/application';

export async function initialize(app: Application): Promise<void> {
  app.deferReadiness();

  window.Sentry = await import('@sentry/ember');

  app.advanceReadiness();
}

export default {
  name: 'load-externals',
  initialize,
};
```

`deferReadiness` / `advanceReadiness` hold the app boot until the dynamic `import()` resolves, so by the time routes mount Sentry is fully wired. The SDK still patches Ember's runtime hooks (router transitions, `ember-concurrency` tasks, components) because that patching happens during `Sentry.init`, not at module import time. `app/sentry.ts` is then dynamically imported alongside the SDK chunk — extend the same initializer with `await import('my-app/sentry')` after the `window.Sentry` assignment, or wire it from your own externals-loading convention.

```js
// ember-cli-build.js — opt out of pieces of the auto-instrumentation
let app = new EmberApp(defaults, {
  sentry: {
    disablePerformance: false,
    disableInstrumentComponents: false,
    ignoreEmberOnerrorPanic: false,
  },
});
```

Wrap every call site behind a thin `sentry` service so the rest of the app never touches `window.Sentry` directly. That keeps the lazy-load contract intact (no static `@sentry/ember` imports outside of `app/sentry.ts` and the initializer) and gives you one chokepoint for swapping providers, stubbing in tests, or adding tags app-wide.

```ts
// app/services/sentry.ts
import Service from '@ember/service';
import type {
  CaptureContext,
  Scope,
  SeverityLevel,
  User,
} from '@sentry/ember';

export default class SentryService extends Service {
  setUser(user: User | null): void {
    window.Sentry.setUser(user);
  }

  captureException(error: unknown, context?: CaptureContext): string {
    return window.Sentry.captureException(error, context);
  }

  captureMessage(message: string, level: SeverityLevel = 'info'): string {
    return window.Sentry.captureMessage(message, level);
  }

  getCurrentScope(): Scope {
    return window.Sentry.getCurrentScope();
  }
}

declare module '@ember/service' {
  interface Registry {
    sentry: SentryService;
  }
}
```

Type-only imports (`import type { ... } from '@sentry/ember'`) are erased by TypeScript and don't pull the SDK into the main bundle — only the runtime calls go through `window.Sentry`, which is populated by the lazy chunk.

```ts
// app/routes/application.ts — identify the user once the session is restored
import Route from '@ember/routing/route';
import { service } from '@ember/service';
import type SessionService from 'ember-simple-auth/services/session';
import type SentryService from 'my-app/services/sentry';

export default class ApplicationRoute extends Route {
  @service declare session: SessionService;
  @service declare sentry: SentryService;

  async beforeModel() {
    await this.session.setup();
    if (this.session.isAuthenticated) {
      const { id, email } = this.session.data.authenticated.user;
      this.sentry.setUser({ id, email });
    } else {
      this.sentry.setUser(null);
    }
  }
}
```

```ts
// app/components/checkout-button.ts — handle errors through the service
import Component from '@glimmer/component';
import { service } from '@ember/service';
import { action } from '@ember/object';
import type SentryService from 'my-app/services/sentry';

export default class CheckoutButton extends Component {
  @service declare sentry: SentryService;

  @action async charge() {
    try {
      await this.args.onCharge();
    } catch (err) {
      this.sentry.captureException(err, { tags: { feature: 'checkout' } });
      throw err;
    }
  }
}
```

```ts
// Add request-scoped context without leaking it across users
this.sentry.getCurrentScope().setContext('order', { id: order.id, total: order.total });
this.sentry.captureMessage('Order submitted with manual override', 'warning');
```

Notes:
- Errors should always be reported via `this.sentry.captureException(...)` — never `import * as Sentry from '@sentry/ember'` at a call site, since that pulls the SDK back into the main bundle and undoes the lazy load. The service is the only sanctioned entry point.
- `ember-concurrency` task errors bubble through `Ember.onerror`, so Sentry hooks them automatically — no manual `try/catch` needed for unhandled task failures.
- Source maps: install `@sentry/cli` and add a `sentry:sourcemaps` script that runs after `ember deploy production` so production stack traces de-minify in the Sentry UI:

  ```json
  // package.json
  {
    "scripts": {
      "sentry:sourcemaps": "sentry-cli sourcemaps inject --org my-org --project my-app ./dist && sentry-cli sourcemaps upload --org my-org --project my-app ./dist"
    }
  }
  ```

  Replace `my-org` / `my-app` with your Sentry org and project slugs. Set `SENTRY_AUTH_TOKEN` in the deploy environment so the CLI can authenticate. The `inject` step embeds debug IDs into the built JS *before* upload — skip it and Sentry can't reliably tie a stack frame to its source map across releases. Wire it as `pnpm ember deploy production && pnpm sentry:sourcemaps` in CI (or your release runbook) so the upload always trails a successful deploy.
- For FastBoot / SSR, install `@sentry/node` in the FastBoot app separately — `@sentry/ember` is browser-only.
- Tune `tracesSampleRate` and `replaysSessionSampleRate` per environment in `config/environment.js`; full sampling in production gets expensive fast.
