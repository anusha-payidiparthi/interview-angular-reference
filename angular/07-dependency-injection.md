# 07 — Dependency Injection & Services

DI is the backbone of Angular. If you get this chapter, the rest of the framework makes sense.

## 1. Concept

A **service** is a class that holds logic or state that isn't tied to one view (API calls,
caching, auth, logging, shared state). Angular's **injector** creates it and hands the same
instance to whoever asks.

| | AngularJS 1.x | React | Angular |
|---|---|---|---|
| Define | `.service()`, `.factory()`, `.provider()`, `.value()`, `.constant()` | module/hook/context | `@Injectable()` class, or `InjectionToken` |
| Get it | Function param names (`function($http)`) — broke on minification | `useContext(X)` / import singleton | `inject(X)` or constructor param type |
| Scope | Always app-wide singleton | Context Provider subtree | Root, route, component subtree — **hierarchical** |
| Swap for tests | `$provide` | Wrap in test Provider / jest.mock | `providers: [{ provide: X, useValue: mock }]` |

## 2. Creating and using a service

```ts
// logger.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })     // one instance for the whole app, tree-shakable
export class Logger {
  log(msg: string) { console.log(`[${new Date().toISOString()}] ${msg}`); }
}
```

### v22: the `@Service()` decorator
```ts
import { Service } from '@angular/core';

@Service()                       // shorthand for @Injectable({ providedIn: 'root' })
export class Logger {
  log(msg: string) { console.log(msg); }
}
```
Use `@Service()` for ordinary app-wide singletons in v22+ code. `@Injectable` is still there (and
is what every pre-v22 codebase uses) for services provided elsewhere (component/route providers)
or needing advanced configuration. The examples in this guide mostly use
`@Injectable({ providedIn: 'root' })` because it works in every version, so read it as `@Service()`
when you're on v22.

```ts
// user.service.ts — a service that depends on other services
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Logger } from './logger.service';

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private logger = inject(Logger);

  getUsers() {
    this.logger.log('fetching users');
    return this.http.get<User[]>('/api/users');
  }
}
```

```ts
// any component
export class UserList {
  private users = inject(UserService);           // modern
  // constructor(private users: UserService) {}  // classic constructor injection — equivalent
}
```

### `inject()` rules
`inject()` only works in an **injection context**: field initializers, the constructor, factory
functions in providers, functional guards/resolvers/interceptors, and `runInInjectionContext`.
It does **not** work inside a method called later (e.g. a click handler) unless you captured the
value earlier.

```ts
// Reusable "hook-like" function — works because it's called from a field initializer
export function injectQueryParam(name: string) {
  const route = inject(ActivatedRoute);
  return toSignal(route.queryParamMap.pipe(map(p => p.get(name))));
}

export class SearchPage {
  q = injectQueryParam('q');    // ✅ injection context
}
```
This is the closest thing Angular has to **custom hooks**.

## 3. Providers: telling the injector *how* to create something

```ts
providers: [
  UserService,                                          // shorthand for { provide: UserService, useClass: UserService }
  { provide: Logger, useClass: ConsoleLogger },         // swap implementation
  { provide: API_URL, useValue: 'https://api.example.com' },  // a value
  { provide: Storage, useFactory: () => (isBrowser() ? localStorage : memoryStorage()) },
  { provide: OldLogger, useExisting: Logger },          // alias: both tokens → same instance
]
```

### `InjectionToken` — for non-class values (config, functions, interfaces)
```ts
import { InjectionToken, inject } from '@angular/core';

export interface AppConfig { apiUrl: string; featureFlags: Record<string, boolean> }
export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// app.config.ts
providers: [{ provide: APP_CONFIG, useValue: { apiUrl: '/api', featureFlags: { beta: true } } }]

// consumer
private config = inject(APP_CONFIG);
```

Token with a built-in default (tree-shakable, no provider needed):
```ts
export const WINDOW = new InjectionToken<Window>('WINDOW', {
  providedIn: 'root',
  factory: () => window,
});
```

### Multi providers — collect many values under one token
```ts
export const VALIDATORS = new InjectionToken<Validator[]>('VALIDATORS');
providers: [
  { provide: VALIDATORS, useClass: EmailValidator, multi: true },
  { provide: VALIDATORS, useClass: PhoneValidator, multi: true },
]
inject(VALIDATORS);   // → [EmailValidator, PhoneValidator]
```
Angular itself uses this for `HTTP_INTERCEPTORS` (legacy), `APP_INITIALIZER`, `NG_VALIDATORS`, etc.

## 4. Hierarchical injectors — the key concept

```
Environment injectors                  Element (node) injectors
─────────────────────                  ────────────────────────
Platform injector                      <app-root>          providers: [...]
   └─ Root injector                       └─ <app-dashboard>  providers: [DashboardState]
        (providedIn: 'root',                    ├─ <app-widget>   inject(DashboardState) → dashboard's instance
         app.config providers)                  └─ <app-widget>   inject(DashboardState) → same instance
         └─ Route injectors
             (route `providers: []`,
              lazy-loaded routes)
```

When a component calls `inject(X)`, Angular walks **up the element tree** looking for a
provider, then up the **environment injectors** to root. The first match wins. No match means an
error (`NullInjectorError: No provider for X`).

### Component-level providers = React Context Provider scoping
```ts
@Injectable()   // no providedIn: must be provided explicitly
export class WizardState {
  step = signal(1);
  data = signal<Partial<Order>>({});
  next() { this.step.update(s => s + 1); }
}

@Component({
  selector: 'app-checkout-wizard',
  providers: [WizardState],          // new instance for EACH <app-checkout-wizard>
  imports: [StepAddress, StepPayment],
  template: `
    @switch (state.step()) {
      @case (1) { <app-step-address /> }
      @case (2) { <app-step-payment /> }
    }
  `,
})
export class CheckoutWizard { protected state = inject(WizardState); }

@Component({ selector: 'app-step-address', template: `<button (click)="state.next()">Next</button>` })
export class StepAddress { protected state = inject(WizardState); }   // gets the wizard's instance
```

React equivalent:
```tsx
const WizardContext = createContext<WizardState | null>(null);
function CheckoutWizard() {
  const state = useWizardState();
  return <WizardContext.Provider value={state}>...</WizardContext.Provider>;
}
```
Differences: no Provider wrapper in the template, and consumers don't re-render just because they
injected it. Only signals they *read* trigger updates.

### `providers` vs `viewProviders`
- `providers`: visible to the component, its view children **and** projected content.
- `viewProviders`: visible to the component's own view only, **not** projected `<ng-content>`.

### Route-level providers
```ts
{ path: 'admin', providers: [AdminService], loadChildren: () => import('./admin/routes') }
```
One instance shared by all components under `/admin`.

## 5. Resolution modifiers

```ts
inject(Logger, { optional: true });   // null instead of error if not found
inject(Logger, { self: true });       // only look in this element's injector
inject(Logger, { skipSelf: true });   // start from the parent (e.g., nested menus finding parent menu)
inject(Logger, { host: true });       // stop at the host component
```
Legacy decorator forms: `@Optional()`, `@Self()`, `@SkipSelf()`, `@Host()`, `@Inject(TOKEN)`.

## 6. Lazy-loaded services with `injectAsync` (v22)

Components and routes could always be lazy-loaded; since v22, **services** can be too. This is
useful for heavy dependencies (PDF/Excel export, charting, rich text) that most users never trigger.

```ts
import { Component, injectAsync } from '@angular/core';

@Component({
  selector: 'app-report',
  template: `<button (click)="export()">Export to PDF</button>`,
})
export class Report {
  // the module isn't downloaded until first call; the service must be auto-provided (@Service())
  private exporter = injectAsync(() => import('./pdf-exporter'));

  async export() {
    const exporter = await this.exporter();
    exporter.export(this.data);
  }
}
```
It also accepts a `prefetch` option (e.g. download on idle). React analogy: `await import('./pdf')`
inside a click handler, except Angular still handles DI and instance creation for you.

## 6b. App initialization

```ts
import { provideAppInitializer, inject } from '@angular/core';

providers: [
  provideAppInitializer(() => inject(ConfigService).load()),   // app waits if it returns a Promise/Observable
]
```
(Legacy: `{ provide: APP_INITIALIZER, useFactory: ..., multi: true }`.)

## 7. Testing benefit

```ts
TestBed.configureTestingModule({
  providers: [{ provide: UserService, useValue: { getUsers: () => of([{ id: 1, name: 'Test' }]) } }],
});
```
No module mocking magic needed. Swap the provider.

## 8. Gotchas

1. **Accidentally multiple instances:** listing a `providedIn: 'root'` service in a component's
   `providers` creates a *separate* instance for that subtree.
2. **Lazy-loaded NgModule providers** (legacy) create child injectors, the source of the classic
   "my singleton isn't a singleton" bug. `providedIn: 'root'` avoids this.
3. **`NullInjectorError: No provider for HttpClient`** means you forgot `provideHttpClient()` in
   `app.config.ts` (or `HttpClientModule` in legacy apps).
4. **Circular dependencies** (A injects B injects A) cause a runtime error. Extract a third service.

## 9. Interview questions

1. **What is DI and why does Angular use it?** A pattern where a class receives its dependencies
   from outside instead of creating them. It gives you loose coupling, easy testing, and
   configurable scope.
2. **What does `providedIn: 'root'` do?** Registers the service in the root injector as a
   tree-shakable singleton.
3. **Explain hierarchical injectors.** Element injectors (from component/directive providers) and
   environment injectors (root, route, lazy). Resolution walks up from the requesting element to
   root, and the first provider found wins.
4. **`useClass` vs `useValue` vs `useFactory` vs `useExisting`?** Instantiate a class; use a given
   value; call a function to build it; alias another token.
5. **What is an `InjectionToken`?** A unique token for injecting non-class values like config or
   interfaces (interfaces don't exist at runtime).
6. **`inject()` vs constructor injection?** Same result. `inject()` works in functions (guards,
   interceptors, reusable helpers), plays well with inheritance and field initializers, and is the
   modern recommendation.
7. **`providers` vs `viewProviders`?** Whether projected content can see the provider.
8. **How do you make a service non-singleton?** Provide it in a component's `providers` (one instance
   per component instance).
9. **What is `@Service()`?** (v22) Shorthand for `@Injectable({ providedIn: 'root' })`.
10. **What is `injectAsync`?** (v22) Asynchronous DI that code-splits a service and loads it on
    first use, with optional prefetching.
