# 00 — High-Level Overview: Modern Angular (v22) at a Glance

Read this first. The goal is a **mental map**, not mastery. Every item links to a deep-dive chapter.
After this, read **[00b — What's new in v21 & v22](./00b-whats-new-latest-versions.md)**.

> **Which version?** The latest is **Angular v22** (June 2026; v22.2 as of Oct 2026). Companies
> run anything from roughly v14 to v22. New projects use v21/v22, and many existing apps are on
> v16–v20 with a mix of old and new syntax. This guide teaches the v22 way and shows the legacy
> syntax so you can work in both.

---

## 1. The one-paragraph summary

Angular 2+ is a **complete rewrite** of AngularJS. It is a **batteries-included, TypeScript-first,
component-based framework**. Where React gives you a view library and lets you choose a router, a
data-fetching library, forms, state and DI, Angular ships **all of them** in one opinionated
package, plus a CLI that generates, builds, tests and upgrades the project.

```
AngularJS 1.x  : MVC-ish, controllers + $scope + directives, 2-way binding, digest cycle, JS
React          : UI library, components = functions, one-way data flow, virtual DOM, JSX
Angular 2+     : Full framework, components = classes + HTML templates, one-way flow by default
                 (2-way opt-in), compiled templates (no virtual DOM), DI everywhere, TypeScript,
                 RxJS + Signals for reactivity
```

---

## 2. Mental model shift

### Coming from AngularJS 1.x — what's GONE

| AngularJS 1.x | Status in Angular 2+ | Replacement |
|---|---|---|
| `$scope` / `$rootScope` | Gone | Component class properties / signals; services for shared state |
| Controllers (`ng-controller`) | Gone | Components |
| `.directive()` with `link`/`compile` | Changed | Components (with template) + Directives (behavior only) |
| Digest cycle / `$apply` / `$watch` | Gone | Change detection (zone.js or zoneless + signals) |
| `$http`, `$q` promises | Replaced | `HttpClient` returning RxJS Observables |
| Filters (`{{ x \| date }}`) | Renamed | **Pipes** (same `\|` syntax) |
| `ng-repeat`, `ng-if`, `ng-show` | Replaced | `@for`, `@if`, `[hidden]` / class binding |
| `ng-model` everywhere | Opt-in | `[(ngModel)]` (template forms) or Reactive Forms |
| `ng-click="fn()"` | Replaced | `(click)="fn()"` |
| `ng-src`, `ng-href`, `ng-class` | Replaced | `[src]`, `[href]`, `[class.x]` — bind to any DOM property |
| `angular.module('app', [...])` | Replaced | ES modules + standalone components (NgModules legacy) |
| `service` / `factory` / `provider` | Unified | `@Injectable()` classes + providers |
| ngRoute / ui-router | Replaced | `@angular/router` |
| String-based DI (`['$http', fn]`) | Gone | Type-based DI (`inject(HttpClient)`) |

### Coming from React — what's DIFFERENT

| React | Angular |
|---|---|
| Component = function that **re-runs** on every render | Component = **class instantiated once**; template is re-checked, the class isn't re-created |
| JSX (JavaScript with HTML inside) | HTML templates with Angular syntax (HTML with expressions inside) |
| Virtual DOM diffing | Compiled templates that update DOM bindings directly (Ivy) |
| `useState`, `useMemo`, `useEffect` | `signal()`, `computed()`, `effect()` — no dependency arrays |
| Props + callback props | `input()` + `output()` |
| Context | Dependency Injection (hierarchical) |
| Hooks for reuse | Services (logic/state), Directives (DOM behavior), Pipes (formatting) |
| Pick your own router/forms/data lib | Built-in Router, Forms, HttpClient |
| `children` | Content projection `<ng-content>` |
| `key` in lists | `track` in `@for` |
| Fragments `<>` | `<ng-container>` |
| CRA/Vite/Next | Angular CLI (`ng`) — esbuild + Vite under the hood |

**The biggest mindset change for a React dev:** your component class **does not re-run**. There
are no stale closures, no dependency arrays, no `useCallback`. State lives on the instance; the
framework figures out what to re-render.

---

## 3. The feature map (everything in one place)

### Language & tooling
| Feature | One-liner | Chapter |
|---|---|---|
| **TypeScript** | Mandatory. Types, classes, decorators, generics. | [01](./01-typescript-essentials.md) |
| **Angular CLI** | `ng new / generate / serve / build / test / update`. | [17](./17-cli-build-ssr.md) |
| **Ivy compiler** | Ahead-of-time (AOT) compiled templates, tree-shakable, small bundles. | [13](./13-change-detection-and-performance.md) |
| **esbuild + Vite builder** | Fast builds/dev server (default since v17). | [17](./17-cli-build-ssr.md) |

### Building UI
| Feature | One-liner | Chapter |
|---|---|---|
| **Components** | Class + template + styles, via `@Component`. | [02](./02-components-and-templates.md) |
| **Standalone components** | No NgModule needed; component declares its own `imports`. Default since v19. | [02](./02-components-and-templates.md) |
| **Templates & binding** | `{{ }}` interpolation, `[prop]`, `(event)`, `[(twoWay)]`. | [03](./03-data-binding-and-control-flow.md) |
| **Built-in control flow** | `@if`, `@for`, `@switch`, `@let` (v17+), replacing `*ngIf`/`*ngFor`. | [03](./03-data-binding-and-control-flow.md) |
| **Deferrable views** | `@defer` lazy-loads part of a template (on viewport, idle, interaction...). | [03](./03-data-binding-and-control-flow.md) |
| **Content projection** | `<ng-content>` ≈ React `children` / slots. | [02](./02-components-and-templates.md) |
| **View encapsulation** | Component CSS is scoped by default (emulated shadow DOM). | [02](./02-components-and-templates.md) |
| **Inputs / Outputs / Model** | `input()`, `output()`, `model()` — signal-based component API. | [04](./04-component-communication.md) |
| **Queries** | `viewChild()`, `contentChild()` ≈ React refs. | [04](./04-component-communication.md) |
| **Lifecycle hooks** | `ngOnInit`, `ngOnDestroy`, `afterNextRender`, `DestroyRef`... | [05](./05-lifecycle-hooks.md) |
| **Error boundaries** | `@boundary { } @error { }` catches errors in part of a template (v22.2, preview). | [03](./03-data-binding-and-control-flow.md) |
| **Directives** | Attribute directives (behavior) and structural directives (DOM shape). | [06](./06-directives-and-pipes.md) |
| **Directive composition** | `hostDirectives` — compose behaviors onto a component (v15+). | [06](./06-directives-and-pipes.md) |
| **Pipes** | Template formatters (`date`, `currency`, `async`, custom). | [06](./06-directives-and-pipes.md) |

### Logic, state & reactivity
| Feature | One-liner | Chapter |
|---|---|---|
| **Dependency Injection** | Hierarchical injectors; `inject()`; `@Service()` for root singletons (v22). | [07](./07-dependency-injection.md) |
| **`injectAsync`** | Lazy-load a service's code on first use (v22). | [07](./07-dependency-injection.md) |
| **Signals** | `signal`, `computed`, `effect`, `linkedSignal`: fine-grained reactivity (v16+, all stable by v20). | [08](./08-signals.md) |
| **resource / httpResource** | Async data as signals, comparable to React Query built in (**stable in v22**). | [08](./08-signals.md), [10](./10-http-client.md) |
| **RxJS** | Observables for streams: HTTP, router events, form changes. | [09](./09-rxjs-and-observables.md) |
| **RxJS interop** | `toSignal`, `toObservable`, `takeUntilDestroyed`. | [08](./08-signals.md) |
| **State management** | Signal services → NgRx SignalStore → NgRx Store (Redux). | [14](./14-state-management.md) |

### Application features
| Feature | One-liner | Chapter |
|---|---|---|
| **HttpClient** | Typed HTTP, interceptors, testing utilities. | [10](./10-http-client.md) |
| **Router** | Nested routes, lazy loading, guards, resolvers, route→input binding. | [11](./11-routing.md) |
| **Forms** | **Signal Forms** (stable v22, new default), Reactive (typed, most common in existing code), Template-driven (`ngModel`). | [12](./12-forms.md) |
| **Change detection** | **Zoneless** (default since v21) + **OnPush by default** (v22); zone.js in older apps. | [13](./13-change-detection-and-performance.md) |
| **SSR & hydration** | Server rendering, hydration, incremental hydration, prerendering. | [17](./17-cli-build-ssr.md) |
| **Testing** | TestBed, component harnesses; Vitest is the default runner for new projects (v21). | [15](./15-testing.md) |
| **NgModules** | Legacy way of grouping code; still in many codebases. | [16](./16-ngmodules-and-architecture.md) |
| **Animations, i18n, PWA, Material/CDK** | Official add-on packages. | [17](./17-cli-build-ssr.md) |

---

## 4. Version timeline (what came when)

Knowing this helps you read older code and answer "what changed recently?" in interviews.

| Version | Year | Headline features |
|---|---|---|
| **2** | 2016 | Rewrite: TypeScript, components, NgModules, DI, RxJS, zone.js, AOT |
| **4** | 2017 | (v3 skipped to align router version) smaller output, `HttpClient` (4.3) |
| **6** | 2018 | `ng update` / `ng add`, `providedIn: 'root'`, Angular Elements, RxJS 6 |
| **8** | 2019 | Differential loading, `import()` lazy routes |
| **9** | 2020 | **Ivy** renderer by default |
| **12–13** | 2021 | Strict mode default, View Engine removed, IE11 dropped |
| **14** | 2022 | **Standalone components** (preview), **typed forms**, `inject()` widely usable |
| **15** | 2022 | Standalone stable, functional guards/interceptors, directive composition, `NgOptimizedImage` |
| **16** | 2023 | **Signals** (preview), required inputs, esbuild (preview), non-destructive hydration |
| **17** | 2023 | **`@if/@for/@switch`**, **`@defer`**, Vite+esbuild default, angular.dev, signals stable |
| **18** | 2024 | Zoneless (experimental), `@let`, event replay, signal inputs/queries/model maturing |
| **19** | 2024 | **Standalone by default**, `linkedSignal`, `resource` (experimental), incremental hydration (preview) |
| **20** | 2025 | `effect`/`linkedSignal`/`toSignal` stable, **zoneless stable**, `httpResource`, new style guide (no `.component` suffix) |
| **21** | Nov 2025 | **Zoneless default** for new apps, **Signal Forms** (experimental), **Vitest default**, Angular Aria (preview) |
| **22** ⭐ | Jun 2026 | **Signal Forms, `resource`/`httpResource`, Angular Aria stable**; **OnPush default** (`Default` → `Eager`); `@Service()`; `injectAsync`; template arrow functions, spread, multi-case `@switch`; webpack builder deprecated; **yearly majors from now on** |
| **22.2** | Sep 2026 | `@boundary` error boundaries & router resources (preview), `private` members usable in templates |
| **23** | ~Jun 2027 | Next major (see <https://angular.dev/reference/releases>) |

**Direction of travel:** NgModules → standalone; zone.js → zoneless; Default CD → OnPush;
RxJS-for-everything → signals for state + RxJS for event streams; decorators (`@Input`) → functions
(`input()`); structural directives (`*ngIf`) → built-in control flow (`@if`); Reactive Forms →
Signal Forms; resolvers → resources.

---

## 5. Architecture at a glance

```
main.ts
  └─ bootstrapApplication(AppComponent, appConfig)
        appConfig.providers:
          provideRouter(routes)            ← routing
          provideHttpClient(withInterceptors(...))  ← HTTP
          provideZonelessChangeDetection() ← change detection mode (v20+)

AppComponent  (root component, <app-root>)
  ├─ <app-header>             ← presentational component
  ├─ <router-outlet>          ← routed pages render here
  │     └─ UsersPageComponent (route: /users, lazy-loaded)
  │           ├─ inject(UserService)   ← service via DI (singleton)
  │           ├─ users = signal<User[]>([])
  │           └─ <app-user-card [user]="u" (select)="onSelect($event)" />
  └─ <app-footer>

UserService (@Injectable({ providedIn: 'root' }))
  └─ inject(HttpClient) → GET /api/users → Observable<User[]> / signal
```

Compared to React:

```
React                           Angular
-----                           -------
main.tsx + createRoot           main.ts + bootstrapApplication
<BrowserRouter><App/>           provideRouter(routes) + <router-outlet>
<QueryClientProvider>           provideHttpClient() + services
<ThemeContext.Provider>         providers: [ThemeService] (DI)
function App() { ... }          @Component class App { ... }
```

---

## 6. A taste of the same component in all three

**AngularJS 1.x**
```js
angular.module('app').component('counter', {
  bindings: { start: '<', onChange: '&' },
  template: `<button ng-click="$ctrl.inc()">Count: {{ $ctrl.count }}</button>`,
  controller: function () {
    this.$onInit = () => { this.count = this.start || 0; };
    this.inc = () => { this.count++; this.onChange({ value: this.count }); };
  }
});
// usage: <counter start="5" on-change="vm.log(value)"></counter>
```

**React**
```tsx
function Counter({ start = 0, onChange }: { start?: number; onChange?: (v: number) => void }) {
  const [count, setCount] = useState(start);
  const double = useMemo(() => count * 2, [count]);
  const inc = () => { const next = count + 1; setCount(next); onChange?.(next); };
  return <button onClick={inc}>Count: {count} (x2 = {double})</button>;
}
// usage: <Counter start={5} onChange={v => console.log(v)} />
```

**Angular (modern)**
```ts
import { Component, computed, input, output, signal, OnInit } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `<button (click)="inc()">Count: {{ count() }} (x2 = {{ double() }})</button>`,
})
export class Counter implements OnInit {
  start = input(0);                 // ≈ prop with default
  changed = output<number>();       // ≈ callback prop

  count = signal(0);                // ≈ useState
  double = computed(() => this.count() * 2); // ≈ useMemo, but no deps array

  ngOnInit() { this.count.set(this.start()); }

  inc() {
    this.count.update(c => c + 1);
    this.changed.emit(this.count());
  }
}
// usage: <app-counter [start]="5" (changed)="log($event)" />
```

Notice: AngularJS's `bindings` became `input()`/`output()`; React's hooks became signals; JSX
became an HTML template with `(event)` and `{{ }}`.

---

## 7. What you'll need most on day one at work

1. Read/write **components** with `input()`, `output()`, signals, and the new control flow.
2. Recognise the **legacy** forms: `@Input()`, `@Output() EventEmitter`, `*ngIf`, `*ngFor`, `NgModule`.
3. Use **services + DI** for API calls and shared state.
4. Be comfortable with basic **RxJS**: `subscribe`, `pipe`, `map`, `switchMap`, `async` pipe.
5. **Routing** with lazy loading and guards.
6. **Reactive forms**.
7. Understand **change detection** and `OnPush`.

## 8. Suggested learning schedule

| Day | Focus |
|---|---|
| 1 | This overview + [00b what's new](./00b-whats-new-latest-versions.md) + chapter 01 (TypeScript) + `ng new` a project |
| 2 | Chapters 02–03 (components, templates, control flow) |
| 3 | Chapters 04–05 (communication, lifecycle) |
| 4 | Chapters 06–07 (directives, pipes, DI) |
| 5 | Chapters 08–09 (signals, RxJS) — the most important pair |
| 6 | Chapters 10–11 (HTTP, routing) |
| 7 | Chapter 12 (forms) |
| 8 | Chapters 13–14 (change detection, state) |
| 9 | Chapters 15–17 (testing, NgModules, CLI/SSR) |
| 10–12 | Chapter 20 mini project, end to end |
| 13–14 | Chapters 18–19 (cheat sheet, interview Qs), mock interviews |
