# 09 — RxJS & Observables

React has no built-in equivalent, which makes this the steepest part of Angular for React devs.
You'll meet RxJS in `HttpClient`, the Router, Reactive Forms and many libraries. You don't
need all 100+ operators; ~15 cover 95% of real code.

## 1. Observable vs Promise

| | Promise | Observable |
|---|---|---|
| Values | One | Zero, one, or many over time |
| Eager/lazy | Eager (runs immediately) | **Lazy**: nothing happens until `subscribe()` |
| Cancellable | No (needs AbortController) | Yes, `unsubscribe()` |
| Operators | `.then` chaining | `pipe(map, filter, switchMap, retry, ...)` |
| AngularJS | `$q` | — |

```ts
import { Observable } from 'rxjs';

const numbers$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  const id = setTimeout(() => { subscriber.next(3); subscriber.complete(); }, 1000);
  return () => clearTimeout(id);          // teardown on unsubscribe
});

const sub = numbers$.subscribe({
  next: v => console.log(v),
  error: e => console.error(e),
  complete: () => console.log('done'),
});
sub.unsubscribe();
```
Convention: a `$` suffix marks an Observable (`users$`).

**Key fact:** `http.get()` returns a *cold* observable. **No request is sent until you
subscribe**, and each subscription sends a new request.

## 2. Creating observables

```ts
import { of, from, interval, timer, fromEvent, EMPTY, throwError } from 'rxjs';

of(1, 2, 3);                          // emits values synchronously, then completes
from([1, 2, 3]);  from(fetch('/x'));  // from array / promise / iterable
interval(1000);                       // 0, 1, 2... every second
timer(500);                           // one value after 500ms
fromEvent<MouseEvent>(document, 'click');
EMPTY;                                // completes immediately
throwError(() => new Error('boom'));
```

## 3. Subjects — observables you can push into

| Type | Behavior | Use |
|---|---|---|
| `Subject<T>` | Multicast, no initial value; late subscribers miss past values | Event bus, triggers |
| `BehaviorSubject<T>` | Has a current value (`.value`), replays latest to new subscribers | State (pre-signals style) |
| `ReplaySubject<T>(n)` | Replays last *n* values | Caches |
| `AsyncSubject<T>` | Emits last value on complete | Rare |

```ts
// Classic (pre-signals) service state pattern — very common in existing codebases
@Injectable({ providedIn: 'root' })
export class CartService {
  private readonly items$$ = new BehaviorSubject<Item[]>([]);
  readonly items$ = this.items$$.asObservable();
  readonly count$ = this.items$.pipe(map(items => items.length));

  add(item: Item) { this.items$$.next([...this.items$$.value, item]); }
}
```
Today you'd write this with signals (chapter 08), but you must be able to read it.

## 4. The operators you actually need

```ts
import { map, filter, tap, switchMap, mergeMap, concatMap, exhaustMap, debounceTime,
         distinctUntilChanged, catchError, retry, take, takeUntil, startWith, shareReplay,
         finalize } from 'rxjs/operators';   // or just from 'rxjs'
import { combineLatest, forkJoin, merge } from 'rxjs';
```

### Transforming & filtering
```ts
users$.pipe(
  map(users => users.filter(u => u.active)),   // transform each emission
  filter(users => users.length > 0),           // drop emissions
  tap(users => console.log('got', users)),     // side effect, value unchanged
);
```

### The four flattening operators (the #1 RxJS interview topic)
When each value triggers *another* observable (usually an HTTP call):

| Operator | Behavior | Use when |
|---|---|---|
| `switchMap` | **Cancel** previous inner, switch to the new one | Search-as-you-type, route param → fetch (latest wins) |
| `mergeMap` | Run all **in parallel** | Independent parallel requests (delete many items) |
| `concatMap` | **Queue**, one after another in order | Order matters (sequential saves) |
| `exhaustMap` | **Ignore** new values while one is running | Prevent double-submit (login button) |

```ts
// typeahead — the canonical example
this.searchControl.valueChanges.pipe(
  debounceTime(300),                // wait for typing pause
  distinctUntilChanged(),           // ignore same value
  filter(q => q.length >= 2),
  switchMap(q => this.api.search(q).pipe(
    catchError(() => of([])),       // keep the stream alive on error
  )),
).subscribe(results => this.results.set(results));

// save button that ignores double clicks
this.saveClicks$.pipe(exhaustMap(() => this.api.save(this.form.value))).subscribe();
```

React comparison: the typeahead in React needs a debounce hook, an AbortController in `useEffect`
cleanup, and a race-condition guard. `switchMap` handles cancellation and races for you.

### Combining streams
```ts
// forkJoin: wait for ALL to complete, then emit once (≈ Promise.all)
forkJoin({ user: api.getUser(id), orders: api.getOrders(id) })
  .subscribe(({ user, orders }) => ...);

// combineLatest: emit whenever ANY emits (after each has emitted once)
combineLatest([filter$, sort$, data$]).pipe(
  map(([filter, sort, data]) => applyFilterSort(data, filter, sort)),
);

// merge: interleave emissions from several streams
merge(clicks$, keypresses$);
```

### Errors
```ts
this.http.get<User>('/api/me').pipe(
  retry({ count: 2, delay: 1000 }),             // retry twice, 1s apart
  catchError(err => {
    this.toast.error('Could not load profile');
    return of(null);                              // fallback value
    // or: return throwError(() => err);           // rethrow
  }),
  finalize(() => this.loading.set(false)),        // always runs (like finally)
);
```
An error **terminates** the stream. Catch errors inside the inner observable (inside `switchMap`)
if the outer stream (like `valueChanges`) must survive.

### Sharing / caching
```ts
// without shareReplay, every subscriber triggers a new HTTP call
readonly countries$ = this.http.get<Country[]>('/api/countries').pipe(
  shareReplay({ bufferSize: 1, refCount: true }),
);
```

### Limiting
```ts
source$.pipe(take(1));               // first value then complete
source$.pipe(first(v => v > 10));    // first matching value
source$.pipe(startWith(initial));    // prepend a value
```

## 5. Subscribing safely (avoiding memory leaks)

Observables that never complete (`interval`, `valueChanges`, router events, subjects) **leak** if
you don't unsubscribe. HTTP observables complete on their own, but unsubscribing still cancels
in-flight requests.

Best → acceptable:
```ts
// 1. Don't subscribe manually: async pipe in template
users$ = this.api.getUsers();
// template: @if (users$ | async; as users) { ... }

// 2. Convert to a signal
users = toSignal(this.api.getUsers(), { initialValue: [] });

// 3. takeUntilDestroyed (in injection context, or pass a DestroyRef)
constructor() {
  this.form.valueChanges.pipe(takeUntilDestroyed()).subscribe(v => this.autosave(v));
}

// 4. Classic: takeUntil + destroy$ subject (very common in older code)
private destroy$ = new Subject<void>();
ngOnInit() { source$.pipe(takeUntil(this.destroy$)).subscribe(...); }
ngOnDestroy() { this.destroy$.next(); this.destroy$.complete(); }

// 5. Manual Subscription
private sub = new Subscription();
ngOnInit() { this.sub.add(source$.subscribe(...)); }
ngOnDestroy() { this.sub.unsubscribe(); }
```

## 6. Hot vs cold

- **Cold:** each subscriber gets its own execution (`http.get`, `of`, `interval`). Like calling a
  function.
- **Hot:** shared execution, and subscribers join mid-stream (`Subject`, `fromEvent`, router events).
  Like a radio broadcast.
- `share()` / `shareReplay()` turns cold into hot.

## 7. The async pipe in templates

```html
@if (vm$ | async; as vm) {
  <h1>{{ vm.user.name }}</h1>
  <p>{{ vm.orders.length }} orders</p>
}
```
```ts
vm$ = combineLatest({ user: this.user$, orders: this.orders$ });   // "view model" pattern
```

## 8. Signals vs RxJS — when to use which

| Use signals | Use RxJS |
|---|---|
| Component/UI state | Event streams (websocket, DOM events) |
| Derived values | Debounce, throttle, buffer, timing |
| Sharing state via services | Cancelling/racing async operations (`switchMap`) |
| Template binding | Complex orchestration (retry/backoff, polling) |
| Simple async with `resource` | Existing APIs that return observables (HttpClient, Router, Forms) |

Bridge with `toSignal` / `toObservable`.

## 9. Interview questions

1. **Observable vs Promise?** See section 1: many values, lazy, cancellable, operators.
2. **`switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`?** Cancel previous / parallel /
   queue / ignore while busy. Give the typeahead and double-submit examples.
3. **Subject vs BehaviorSubject?** BehaviorSubject requires an initial value, holds the current
   value, and emits it to new subscribers.
4. **Hot vs cold observables?** Cold creates a new producer per subscriber; hot shares one producer.
5. **How do you avoid memory leaks?** `async` pipe, `toSignal`, `takeUntilDestroyed`, `takeUntil`
   with a destroy subject, or unsubscribe in `ngOnDestroy`.
6. **Why doesn't my HTTP request fire?** Nobody subscribed. Observables are lazy.
7. **Why does my request fire twice?** Two subscribers (e.g. two `async` pipes). Use `shareReplay`
   or a single subscription.
8. **`forkJoin` vs `combineLatest`?** `forkJoin` emits once when all complete (`Promise.all`);
   `combineLatest` emits on every change once all have emitted.
9. **What happens when an observable errors?** It terminates. Use `catchError` to recover, inside
   the flattening operator if the outer stream must continue.
