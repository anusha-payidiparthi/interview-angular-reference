# 08 — Signals

Signals are Angular's modern reactivity primitive (introduced v16, core APIs stable by v20). For a
React developer they're the most natural part of modern Angular: think **`useState` +
`useMemo` + `useEffect` without dependency arrays or re-running components.**

## 1. The three core primitives

```ts
import { signal, computed, effect } from '@angular/core';

const count = signal(0);                       // WritableSignal<number>
const double = computed(() => count() * 2);    // Signal<number> — read-only, memoized, lazy

effect(() => console.log('count is', count(), 'double is', double()));

count();             // read → 0  (signals are getter functions)
count.set(5);        // replace
count.update(c => c + 1);   // derive from previous (like setState(prev => ...))
```

| React | Angular signal | Notes |
|---|---|---|
| `const [x, setX] = useState(0)` | `x = signal(0)` | read `x()`, write `x.set()` / `x.update()` |
| `useMemo(() => a * b, [a, b])` | `computed(() => a() * b())` | deps tracked automatically, never stale |
| `useEffect(() => {...}, [a])` | `effect(() => {...a()...})` | runs when any read signal changes |
| `useRef` (mutable, no re-render) | plain class field | the class isn't re-created anyway |
| `useCallback` | not needed | methods are stable on the instance |
| Context + useContext | service with signals + `inject()` | see chapter 07 |

AngularJS comparison: `computed` is a `$watch` that recalculates only when its dependencies
change, without a digest cycle and without dirty-checking everything.

## 2. How it works (the mental model)

- Reading a signal **inside a reactive context** (template, `computed`, `effect`) registers a
  dependency.
- When the signal changes, dependents are **notified**. `computed` values are marked dirty and
  recompute **lazily** the next time they're read. Templates that read the signal are scheduled for
  re-render. **Only the components that read it** are updated, not the whole tree.
- Signals use `Object.is` equality by default: setting the same value notifies no one.

```ts
const user = signal({ name: 'Ana', age: 30 });

user().age = 31;                                  // ❌ mutation — nobody is notified
user.update(u => ({ ...u, age: 31 }));            // ✅ new reference — notifies

// custom equality
const pos = signal({ x: 0, y: 0 }, { equal: (a, b) => a.x === b.x && a.y === b.y });
```
Same immutability rule as React state.

## 3. Signals in a component

```ts
import { Component, computed, signal } from '@angular/core';

interface Todo { id: number; title: string; done: boolean }
type Filter = 'all' | 'active' | 'done';

@Component({
  selector: 'app-todos',
  template: `
    <input #box (keyup.enter)="add(box.value); box.value = ''" placeholder="What needs doing?" />

    @for (f of filters; track f) {
      <button [class.active]="filter() === f" (click)="filter.set(f)">{{ f }}</button>
    }

    <ul>
      @for (t of visible(); track t.id) {
        <li>
          <input type="checkbox" [checked]="t.done" (change)="toggle(t.id)" />
          {{ t.title }}
        </li>
      }
    </ul>
    <p>{{ remaining() }} left</p>
  `,
})
export class Todos {
  protected readonly filters: Filter[] = ['all', 'active', 'done'];
  protected todos = signal<Todo[]>([]);
  protected filter = signal<Filter>('all');

  protected visible = computed(() => {
    const f = this.filter();
    return this.todos().filter(t => f === 'all' || (f === 'done' ? t.done : !t.done));
  });
  protected remaining = computed(() => this.todos().filter(t => !t.done).length);

  add(title: string) {
    if (!title.trim()) return;
    this.todos.update(ts => [...ts, { id: Date.now(), title, done: false }]);
  }

  toggle(id: number) {
    this.todos.update(ts => ts.map(t => (t.id === id ? { ...t, done: !t.done } : t)));
  }
}
```

## 4. `effect()` — side effects only

```ts
export class Settings {
  theme = signal<'light' | 'dark'>(
    (localStorage.getItem('theme') as 'light' | 'dark') ?? 'light'
  );

  constructor() {
    effect(() => {
      localStorage.setItem('theme', this.theme());        // sync to storage
      document.body.classList.toggle('dark', this.theme() === 'dark');
    });
  }
}
```

Rules:
- Create effects in an **injection context** (constructor/field), or pass `{ injector }`.
- They're automatically destroyed with the component/service.
- Use them for **syncing to the outside world**: logging, localStorage, DOM APIs, third-party libs.
- **Don't use effects to derive state.** Use `computed` or `linkedSignal`. (Same advice as
  React's "you might not need an effect".)
- Effects run asynchronously, during change detection, not synchronously on `set()`.

### `untracked()` — read without subscribing
```ts
effect(() => {
  const user = this.currentUser();                 // tracked
  untracked(() => this.analytics.log('user changed', this.sessionId())); // sessionId NOT tracked
});
```

## 5. `linkedSignal()` — writable state that resets from a source

Useful for "local copy of an input" or "selection that resets when the list changes". In React
you'd do this with a `key` reset or `useEffect` + `setState`.

```ts
export class ShippingPicker {
  options = input.required<string[]>();
  // defaults to the first option; user can change it; resets if options change
  selected = linkedSignal(() => this.options()[0]);

  choose(o: string) { this.selected.set(o); }
}

// advanced form: keep previous selection if it still exists
selected = linkedSignal<string[], string>({
  source: this.options,
  computation: (opts, prev) => (prev && opts.includes(prev.value) ? prev.value : opts[0]),
});
```

## 6. Async data as signals: `resource()` and `httpResource()` (stable since v22)

```ts
import { resource, signal, computed } from '@angular/core';

export class UserDetail {
  userId = input.required<number>();

  user = resource({
    params: () => ({ id: this.userId() }),          // re-runs when userId changes (was `request` in v19)
    loader: async ({ params, abortSignal }) => {
      const res = await fetch(`/api/users/${params.id}`, { signal: abortSignal });  // previous request auto-aborted
      return (await res.json()) as User;
    },
  });
}
```
```html
@if (user.isLoading()) { <app-spinner /> }
@else if (user.error()) { <p>Failed to load</p> }
@else if (user.hasValue()) { <h2>{{ user.value().name }}</h2> }
<button (click)="user.reload()">Refresh</button>
```
That's roughly React Query's `useQuery` built in. `httpResource` (chapter 10) does the same with
`HttpClient`, so interceptors apply. For Observable-based loaders there's `rxResource` (from
`@angular/core/rxjs-interop`, with a `stream` instead of a `loader`).

Status history: experimental in v19/v20, **production-ready in v22**. Older code you'll see
uses `request` instead of `params` (renamed in v20).

## 7. RxJS interop

```ts
import { toSignal, toObservable, takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class Search {
  private http = inject(HttpClient);
  private route = inject(ActivatedRoute);

  query = signal('');

  // Observable → Signal (subscribes immediately, unsubscribes on destroy)
  id = toSignal(this.route.paramMap.pipe(map(p => p.get('id'))), { initialValue: null });

  // Signal → Observable → use RxJS operators → back to Signal
  results = toSignal(
    toObservable(this.query).pipe(
      debounceTime(300),
      distinctUntilChanged(),
      switchMap(q => (q ? this.http.get<Result[]>(`/api/search?q=${q}`) : of([]))),
    ),
    { initialValue: [] },
  );
}
```

**Rule of thumb:** signals for **state** ("what is the value now?"); RxJS for **events over
time** ("what happens when things arrive?": debouncing, cancellation, websockets, combining
streams).

## 8. Signal-based component APIs (recap)

```ts
name = input<string>();             // Signal<string | undefined>
id = input.required<number>();      // Signal<number>
value = model(0);                   // WritableSignal + valueChange output
saved = output<Item>();             // not a signal, but same function style
el = viewChild<ElementRef>('ref');  // Signal<ElementRef | undefined>
items = contentChildren(Item);      // Signal<readonly Item[]>
```

## 9. Read-only exposure pattern (services)

```ts
@Injectable({ providedIn: 'root' })
export class AuthStore {
  private readonly _user = signal<User | null>(null);
  readonly user = this._user.asReadonly();              // consumers can't .set()
  readonly isLoggedIn = computed(() => this._user() !== null);

  login(u: User) { this._user.set(u); }
  logout() { this._user.set(null); }
}
```

## 10. Gotchas

1. **Forgetting `()`** in templates: `{{ count }}` prints the function source; you need `{{ count() }}`.
   With `[(x)]` two-way binding, pass the signal **without** `()`.
2. **Mutating objects/arrays** inside a signal doesn't notify. Always produce new references.
3. **Setting signals inside `computed`** isn't allowed. `computed` must be pure.
4. **Effect loops:** an effect that writes a signal it also reads can loop. Prefer `computed`, or
   use `untracked`.
5. **Conditional reads:** `computed(() => flag() ? a() : b())` only tracks `b` when `flag` is false.
   That's correct and efficient, but surprising the first time.
6. **`toSignal` without `initialValue`** gives `T | undefined`. Use `{ requireSync: true }` for
   synchronous sources like `BehaviorSubject`.

## 11. Interview questions

1. **What are signals?** Reactive value wrappers that notify consumers when they change, enabling
   fine-grained change detection without zone.js.
2. **`signal` vs `computed` vs `effect`?** Writable state; derived read-only memoized state; side
   effects that re-run on dependency change.
3. **Signals vs Observables?** Signals always have a current value, are synchronous to read, and
   track dependencies automatically. Observables are push-based streams over time, can be cold or
   lazy, need subscription, and come with rich operators. They're complementary.
4. **Why are signals good for performance?** Angular knows exactly which views depend on which
   values, so it can update only those (and go zoneless).
5. **What's `linkedSignal`?** A writable signal whose value resets from a computation whenever its
   source changes.
6. **How do you convert between signals and observables?** `toSignal()` and `toObservable()` from
   `@angular/core/rxjs-interop`.
7. **When would you NOT use an effect?** For deriving state (use `computed`/`linkedSignal`) or for
   propagating state between signals.
