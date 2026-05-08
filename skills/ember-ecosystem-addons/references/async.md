# Async / orchestration

## [`ember-concurrency`](http://ember-concurrency.com)

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
