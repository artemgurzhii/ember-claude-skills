# Authentication

## [`ember-simple-auth`](https://github.com/mainmatter/ember-simple-auth)

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

## [`ember-cookies`](https://github.com/mainmatter/ember-cookies)

Read / write cookies from anywhere — components, services, route hooks, FastBoot. The standard companion to `ember-simple-auth` when you want a cookie-backed session store, and the easy answer for any "I need to read a cookie" task that would otherwise hand-roll `document.cookie`.

```bash
ember install ember-cookies
```

```ts
import Service, { service } from '@ember/service';
import type CookiesService from 'ember-cookies/services/cookies';

export default class ConsentService extends Service {
  @service declare cookies: CookiesService;

  get accepted() {
    return this.cookies.read('cookie-consent') === 'yes';
  }

  accept() {
    this.cookies.write('cookie-consent', 'yes', { maxAge: 60 * 60 * 24 * 365, path: '/' });
  }
}
```

Works in FastBoot — same API on server and browser, so SSR-rendered pages can read the request cookie without branching.
