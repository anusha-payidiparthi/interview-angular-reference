# 05 — Lifecycle Hooks

## 1. The full list, in order

| Hook | When | Typical use | React rough equivalent | AngularJS 1.x |
|---|---|---|---|---|
| `constructor` | Class instantiated | `inject()` dependencies, set up signals/effects. **Inputs not set yet** | function body (first render) | controller fn |
| `ngOnChanges(changes)` | Before `ngOnInit` and whenever an `@Input` reference changes | React to legacy input changes | `useEffect(..., [prop])` | `$onChanges` |
| `ngOnInit` | Once, after first `ngOnChanges` | Initialization that needs inputs (fetch data) | `useEffect(..., [])` | `$onInit` |
| `ngDoCheck` | Every change detection run | Custom change detection (rare) | — | `$doCheck` |
| `ngAfterContentInit` | Once, after projected content is initialized | Work with `contentChild` | — | — |
| `ngAfterContentChecked` | After every check of projected content | Rare | — | — |
| `ngAfterViewInit` | Once, after the component's view + child views render | Access DOM / `viewChild` | `useLayoutEffect(..., [])` | `$postLink` |
| `ngAfterViewChecked` | After every check of the view | Rare | — | — |
| `ngOnDestroy` | Just before destruction | Cleanup: unsubscribe, timers, listeners | `useEffect` cleanup | `$onDestroy` |

Implement the matching interface for type safety: `implements OnInit, OnDestroy`.

## 2. Example with the classic hooks

```ts
import { Component, Input, OnChanges, OnInit, OnDestroy, AfterViewInit, SimpleChanges, ElementRef, ViewChild } from '@angular/core';
import { interval, Subscription } from 'rxjs';

@Component({
  selector: 'app-clock',
  template: `<canvas #canvas></canvas> {{ now }}`,
})
export class Clock implements OnChanges, OnInit, AfterViewInit, OnDestroy {
  @Input() timezone = 'UTC';
  @ViewChild('canvas') canvas!: ElementRef<HTMLCanvasElement>;
  now = '';
  private sub?: Subscription;

  ngOnChanges(changes: SimpleChanges) {
    if (changes['timezone'] && !changes['timezone'].firstChange) {
      console.log('timezone changed to', this.timezone);
    }
  }

  ngOnInit() {
    this.sub = interval(1000).subscribe(() => {
      this.now = new Date().toLocaleTimeString('en-US', { timeZone: this.timezone });
    });
  }

  ngAfterViewInit() {
    const ctx = this.canvas.nativeElement.getContext('2d');   // DOM is available now
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();       // avoid memory leaks
  }
}
```

## 3. The modern way: fewer hooks

With signals and the newer APIs, you rarely need `ngOnChanges`, `ngAfterViewInit` or `ngOnDestroy`:

```ts
import { Component, DestroyRef, ElementRef, afterNextRender, computed, effect, inject, input, signal, viewChild } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { interval } from 'rxjs';

@Component({
  selector: 'app-clock',
  template: `<canvas #canvas></canvas> {{ now() }}`,
})
export class Clock {
  timezone = input('UTC');
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');
  private tick = signal(new Date());

  // replaces ngOnChanges + manual recomputation
  now = computed(() => this.tick().toLocaleTimeString('en-US', { timeZone: this.timezone() }));

  constructor() {
    // replaces ngOnInit subscribe + ngOnDestroy unsubscribe
    interval(1000).pipe(takeUntilDestroyed()).subscribe(() => this.tick.set(new Date()));

    // replaces ngAfterViewInit for DOM work (runs once, browser only — SSR safe)
    afterNextRender(() => {
      const ctx = this.canvas().nativeElement.getContext('2d');
    });

    // react to input changes with side effects (like useEffect with deps, minus the deps array)
    effect(() => console.log('timezone is now', this.timezone()));

    // generic cleanup
    inject(DestroyRef).onDestroy(() => console.log('destroyed'));
  }
}
```

| Old | New |
|---|---|
| `ngOnChanges` to derive state | `computed()` on `input()` signals |
| `ngOnChanges` for side effects | `effect()` |
| `ngAfterViewInit` for DOM | `afterNextRender()` (once) / `afterEveryRender()` (every render) |
| `ngOnDestroy` + `Subscription` | `takeUntilDestroyed()` or `DestroyRef.onDestroy()` |
| `ngOnInit` | Still fine! Use it when you need input values at init. |

> `afterRender` was renamed **`afterEveryRender`** in v20. `afterNextRender`/`afterEveryRender`
> also accept phases (`{ earlyRead, write, mixedReadWrite, read }`) to avoid layout thrashing.

## 4. Constructor vs `ngOnInit`

- **Constructor:** runs when the class is created. Use it for DI (`inject()`) and wiring up
  signals/effects. **Input values are not available yet** (legacy `@Input` properties hold defaults;
  reading an `input()` signal that's required throws an error).
- **`ngOnInit`:** inputs are set. Classic place to kick off data loading based on inputs.

Interviewers love this question.

## 5. Mapping React's `useEffect` mental model

```tsx
// React
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);       // cleanup
}, []);

useEffect(() => { document.title = name; }, [name]);
```
```ts
// Angular
constructor() {
  const id = setInterval(() => this.tick(), 1000);
  inject(DestroyRef).onDestroy(() => clearInterval(id));

  effect(() => { document.title = this.name(); });   // deps tracked automatically
}
```
With effect cleanup (like returning a function from `useEffect`):
```ts
effect((onCleanup) => {
  const id = setTimeout(() => this.save(this.draft()), 500);   // debounce autosave
  onCleanup(() => clearTimeout(id));
});
```

## 6. Gotchas

1. **`ExpressionChangedAfterItHasBeenCheckedError`** (dev mode): you changed a bound value *during*
   change detection, typically in `ngAfterViewInit`. Fix it by deriving with `computed`, moving the
   logic earlier (`ngOnInit`), or using signals. Don't "fix" it with `setTimeout` unless you must.
2. **`ngOnChanges` only fires for `@Input` reference changes** (from the parent's template), not for
   internal mutations or object property changes.
3. **Hooks on services:** only `ngOnDestroy` works on `@Injectable` classes.
4. **SSR:** `afterNextRender`/`afterEveryRender` don't run on the server, so they're the right place
   for `window`/`document` access.

## 7. Interview questions

1. **List the lifecycle hooks in order.** constructor → ngOnChanges → ngOnInit → ngDoCheck →
   ngAfterContentInit → ngAfterContentChecked → ngAfterViewInit → ngAfterViewChecked → ngOnDestroy.
2. **Constructor vs ngOnInit?** See section 4.
3. **When is `@ViewChild` available?** In `ngAfterViewInit`, or in `ngOnInit` with `static: true`.
4. **How do you prevent memory leaks from subscriptions?** `async` pipe, `takeUntilDestroyed()`,
   `toSignal()`, or unsubscribing in `ngOnDestroy`.
5. **What's `DestroyRef`?** An injectable that lets you register destroy callbacks from anywhere in
   an injection context, including outside the class (reusable functions).
