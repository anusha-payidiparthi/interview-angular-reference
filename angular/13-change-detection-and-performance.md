# 13 — Change Detection & Performance

## 1. The core question: "how does the UI know to update?"

| | Mechanism |
|---|---|
| **AngularJS 1.x** | **Digest cycle**: after events wrapped in `$apply`, loop over *all* `$watch`ers, compare old vs new, repeat until stable (max 10 iterations). Outside Angular (setTimeout, jQuery) → call `$scope.$apply()` manually. |
| **React** | Explicit: `setState` schedules a re-render of that component **and its children**; the virtual DOM diff decides what DOM to touch. `memo` / `useMemo` to skip work. |
| **Angular (zone.js)** | **zone.js** monkey-patches async APIs (events, timers, XHR, promises). After any async task, Angular runs change detection **top-down over the component tree**, comparing each template binding's previous value. No virtual DOM; bindings update the DOM directly. |
| **Angular (zoneless + signals)** | No monkey-patching. Change detection is scheduled only when something tells Angular: a **signal read by a template changes**, a template event handler fires, `markForCheck()`, `async` pipe emission, input change. Default for new apps from v21; combined with OnPush-by-default in v22. |

## 2. Eager (formerly "Default") vs OnPush

> **v22 change:** `OnPush` is now the **default** when `changeDetection` is omitted, and the old
> `ChangeDetectionStrategy.Default` was renamed **`ChangeDetectionStrategy.Eager`**. In code before
> v22 you'll see `changeDetection: ChangeDetectionStrategy.OnPush` written explicitly on every
> component (best practice back then). `ng update` to v22 marks components that relied on the old
> default as `Eager` so behavior doesn't change.

### Eager strategy (pre-v22 default, called `Default`)
Every change detection pass checks **every component** in the tree. Simple, but it's work
proportional to app size.

### `OnPush` strategy
```ts
@Component({ changeDetection: ChangeDetectionStrategy.OnPush, ... })
```
The component (and its subtree) is checked **only when**:
1. An **input reference** changes (`===` comparison, not deep).
2. An **event handler** in its template (or children's) fires.
3. A **signal read in its template** changes.
4. An **`async` pipe** in its template receives a value.
5. You call **`ChangeDetectorRef.markForCheck()`** manually.

Think of OnPush as `React.memo` for the whole subtree, except that signals and events still punch
through precisely.

```ts
// ❌ Bug with OnPush: mutating the input object — child doesn't update
this.user.name = 'New';
// ✅ New reference
this.user = { ...this.user, name: 'New' };
// ✅✅ Signals — mutations are explicit and tracked
this.user.update(u => ({ ...u, name: 'New' }));
```

**Recommendation:** `OnPush` on all components combined with signals. In v22 you get it
automatically; in older versions add it explicitly. This is the modern performance baseline and
is zoneless-ready.

### Manual control (rarely needed with signals)
```ts
private cdr = inject(ChangeDetectorRef);
this.cdr.markForCheck();   // mark this + ancestors dirty for next pass (OnPush)
this.cdr.detectChanges();  // run CD now for this component subtree
this.cdr.detach();         // opt out entirely (e.g., high-frequency charts), reattach() later
```

## 3. zone.js vs zoneless

```ts
// v20+: opt in (v21 new projects: already default)
providers: [provideZonelessChangeDetection()]
// and remove 'zone.js' from polyfills in angular.json
```

Benefits: smaller bundle, no monkey-patching, better stack traces, works better with native
async/await, faster. Requirements: state changes must be signals, `async` pipe, `markForCheck`,
or template events. A plain `setTimeout(() => this.x = 5)` **won't re-render** zoneless (it would
with zone.js). Use `this.x.set(5)` with a signal.

Running code outside Angular in zone-based apps (to avoid triggering CD for, e.g., mousemove):
```ts
private zone = inject(NgZone);
this.zone.runOutsideAngular(() => window.addEventListener('mousemove', this.track));
this.zone.run(() => this.updateUi());   // re-enter when you need CD
```

## 4. `ExpressionChangedAfterItHasBeenCheckedError`

In dev mode, Angular runs a second verification pass. If a binding's value changed between the
passes, you get this error. It means **one-way data flow was violated**: something updated
parent state *during* rendering (e.g. a child setting a parent value in `ngAfterViewInit`).

Fixes (best first): derive with `computed`; move the logic earlier (`ngOnInit`, constructor);
restructure so the data owner sets it; as a last resort, defer with
`queueMicrotask`/`setTimeout` plus a signal.

React analog: "Cannot update a component while rendering a different component".

## 5. Performance checklist

### Rendering
- `ChangeDetectionStrategy.OnPush` everywhere + **signals**.
- **`track`** in `@for` with stable ids (wrong tracking re-creates DOM).
- Avoid **method calls in templates** that do work (`{{ calculateTotal() }}`). Use `computed` or pure pipes.
- **Pure pipes** for formatting.
- **Virtual scrolling** for long lists: `@angular/cdk/scrolling` → `<cdk-virtual-scroll-viewport>`.
- Run high-frequency non-UI work outside Angular (zone apps) or with no signal writes.

### Loading
- **Lazy-load routes** (`loadComponent`, `loadChildren`).
- **`@defer`** for below-the-fold or heavy widgets (charts, editors, comments).
- **Preloading** strategy for likely next routes.
- **`NgOptimizedImage`**: `<img ngSrc="hero.jpg" width="800" height="400" priority />` gives lazy
  loading, `srcset`, LCP hints, and CLS warnings.
- **Bundle budgets** in `angular.json` fail the build if bundles grow too large.
- `ng build --stats-json` + `esbuild` analyzer / `source-map-explorer` to inspect bundles.

### Server
- **SSR + hydration** for faster first paint and SEO; **incremental hydration** with `@defer (hydrate on viewport)`.
- **Prerendering (SSG)** for static routes.
- HTTP **transfer cache** (automatic with `provideClientHydration()`) avoids re-fetching on the client.

### Memory
- Unsubscribe (`takeUntilDestroyed`, `async` pipe, `toSignal`).
- Remove global listeners in `DestroyRef.onDestroy`.

### Tooling
- **Angular DevTools** (browser extension): component tree, profiler, injector tree, signal graph.
- Chrome Performance panel.

## 6. How Ivy compiles templates (for the curious / senior interviews)

The AOT compiler turns each template into **instructions** (create DOM nodes, then update
bindings). At runtime, change detection calls the "update" part for each checked component and
compares each binding's new value with the stored previous value; only changed bindings touch the
DOM. There's **no virtual DOM tree to diff**, which uses less memory. Ivy also enables
tree-shaking (unused framework features are dropped) and locality (each component compiles
independently).

## 7. Interview questions

1. **How does change detection work in Angular?** zone.js notifies Angular after async tasks;
   Angular checks component templates top-down, comparing bindings and updating changed DOM. With
   signals/zoneless, CD is scheduled by signal changes and events, and can target specific views.
2. **Eager (Default) vs OnPush?** Eager checks every component each cycle; OnPush checks only on new
   input references, template events, signal changes, async pipe emissions, or markForCheck.
   OnPush is the default in v22, and `Default` was renamed `Eager`.
3. **What is zone.js? Why go zoneless?** A library that patches async APIs so Angular knows when to
   run CD. Zoneless removes the overhead and the magic: smaller bundles, better performance and
   debugging, relying on signals for notifications.
4. **`markForCheck` vs `detectChanges`?** `markForCheck` flags the component and its ancestors to be
   checked in the next cycle; `detectChanges` synchronously checks this component's subtree now.
5. **What causes ExpressionChangedAfterItHasBeenCheckedError?** Changing a bound value after it was
   checked in the same cycle (violating unidirectional flow). It only shows in dev mode.
6. **How would you optimize a slow Angular app?** Section 5: OnPush+signals, track, lazy
   loading/defer, virtual scroll, avoid template method calls, SSR, bundle analysis, and profile
   with DevTools.
7. **Angular vs React rendering?** React re-runs component functions and diffs a virtual DOM.
   Angular compiles templates to instructions and checks bindings directly. With signals, Angular
   can skip unaffected components without memoization.
8. **How did AngularJS's digest cycle differ?** Dirty-checking all watchers repeatedly until stable,
   with two-way binding causing cascades. Angular 2+ has unidirectional flow and a single pass per
   cycle (plus a dev-mode verification pass).
