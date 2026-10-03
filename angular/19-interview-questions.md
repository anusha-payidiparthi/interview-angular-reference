# 19 — Angular Interview Questions & Answers

Grouped by topic, roughly from fundamentals to senior level. Practice answering **out loud** in
2–4 sentences, then give an example. Where useful, contrast with React, since interviewers like
hearing that you understand both.

---

## A. Fundamentals

**1. What is Angular and how is it different from AngularJS?**
Angular (2+) is a TypeScript-based, component-driven framework, a full rewrite of AngularJS. It
replaced controllers/`$scope` with components, the digest cycle with change detection (zone.js or
signals), string-based DI with type-based hierarchical DI, and added a CLI, AOT compilation,
RxJS, and a modular router.

**2. Angular vs React?**
Angular is a full framework (router, forms, HTTP, DI, testing, CLI built in, opinionated);
React is a UI library where you choose the rest. Angular uses HTML templates compiled AOT with no
virtual DOM; React uses JSX and VDOM diffing. Angular components are classes instantiated once;
React components are functions that re-run. State: signals vs hooks.

**3. What are the main building blocks of an Angular app?**
Components, templates, directives, pipes, services + DI, routing, and (legacy) NgModules,
bootstrapped with `bootstrapApplication` and configured with providers.

**4. What's the latest Angular version and what's notable?**
v22 (June 2026; v22.2 current). Stable Signal Forms, `resource`/`httpResource`, Angular Aria;
OnPush default (`Default` renamed `Eager`); `@Service()`; `injectAsync`; template arrow
functions, spread, multi-case `@switch`; `@boundary` error boundaries (preview); yearly majors.

**5. What is TypeScript's role?**
Angular is written in TS. Types enable DI by type, strict template type-checking, decorators for
metadata, and better tooling and refactoring.

**6. What is a decorator? Which does Angular use?**
A function adding metadata to classes/members: `@Component`, `@Directive`, `@Pipe`,
`@Injectable`, `@Service`, `@NgModule`; legacy member decorators `@Input`, `@Output`,
`@ViewChild`, `@HostListener`, `@HostBinding`.

**7. What is AOT compilation?**
Templates are compiled to JavaScript at build time, which means faster rendering, smaller bundles
(no compiler shipped), template errors caught at build, and better security (no runtime template
evaluation).

**8. What is Ivy?**
Angular's compilation and rendering engine (default since v9). Templates compile to instruction
functions, giving tree-shaking, smaller bundles, and component-level compilation (locality).

## B. Components & templates

**9. What is a component?** A class with `@Component` that controls a view: selector, template,
styles, imports, providers, change detection strategy.

**10. Standalone components?** Components that declare their template dependencies in `imports`
without an NgModule. Default since v19.

**11. Four types of data binding?** Interpolation `{{ }}`, property `[x]`, event `(x)`, two-way `[(x)]`.

**12. How does `[(ngModel)]` work?** Sugar for `[ngModel]="v" (ngModelChange)="v = $event"`.
Generally, `[(x)]` = `[x]` + `(xChange)`.

**13. Property vs attribute binding?** Property binding sets DOM properties; `[attr.x]` sets HTML
attributes (for aria, colspan, svg).

**14. What is content projection?** Passing content into a child's `<ng-content>`; multi-slot
with `select`. It's the equivalent of React `children`.

**15. `ng-template` vs `ng-container` vs `ng-content`?** `ng-template`: a lazy template, not
rendered by itself. `ng-container`: a grouping element with no DOM node. `ng-content`: the
projection slot.

**16. View encapsulation modes?** Emulated (default, attribute-scoped CSS), ShadowDom, None.

**17. Template reference variable?** `#name` gives the template access to an element, component
or directive instance.

**18. New control flow vs structural directives?** `@if/@for/@switch` are built in: no imports,
better type narrowing, `@empty`, required `track`, faster. Migration schematic available.

**19. Why is `track` required in `@for`?** Identity for DOM reuse when lists change (React `key`).

**20. What is `@defer`?** Lazy-loads a template block's dependencies as a separate chunk based on
triggers (idle, viewport, interaction, hover, timer, when), with placeholder/loading/error blocks.
Also drives incremental hydration.

**21. What is `@let`?** Declares a read-only local template variable.

**22. What is `@boundary`?** (v22.2 preview) A template error boundary: if content throws, it
renders the `@error` block instead of breaking the page.

## C. Component communication

**23. How do components communicate?** `input()`/`output()`, `model()`, `viewChild`/`contentChild`,
shared services (signals/RxJS), router state.

**24. `input()` vs `@Input()`?** `input()` returns a read-only signal, so you derive values with
`computed` and need no `ngOnChanges`. It supports `required`, `transform`, `alias`.

**25. What is `model()`?** A writable input signal that also emits `<name>Change`, enabling
two-way binding.

**26. `ViewChild` vs `ContentChild`?** Own template vs projected content.

**27. How does a child talk to a parent?** `output()` emits; the parent binds `(event)`.

**28. How do siblings communicate?** Via a shared service (root or common-ancestor provider).

## D. Lifecycle

**29. Lifecycle hooks in order?** constructor → ngOnChanges → ngOnInit → ngDoCheck →
ngAfterContentInit → ngAfterContentChecked → ngAfterViewInit → ngAfterViewChecked → ngOnDestroy.

**30. Constructor vs `ngOnInit`?** The constructor is for DI and wiring, and inputs aren't set yet.
`ngOnInit` runs once inputs are available.

**31. What replaces `ngAfterViewInit` for DOM work?** `afterNextRender()` / `afterEveryRender()`,
which are SSR-safe.

**32. How to clean up?** `DestroyRef.onDestroy`, `takeUntilDestroyed`, `ngOnDestroy`.

## E. Directives & pipes

**33. Types of directives?** Components, attribute directives, structural directives.

**34. How do you create a structural directive?** Inject `TemplateRef` + `ViewContainerRef`, call
`createEmbeddedView` / `clear` based on an input.

**35. What is the directive composition API?** `hostDirectives` composes directives onto a
component/directive, optionally exposing their inputs/outputs.

**36. Pure vs impure pipes?** Pure pipes recompute when input references change; impure pipes
recompute every CD cycle.

**37. What does the `async` pipe do?** Subscribes, returns the latest value, triggers CD, and
unsubscribes on destroy.

## F. Dependency injection

**38. What is DI?** Classes receive dependencies from an injector rather than creating them. You
get decoupling, testability, and configurable scope.

**39. `providedIn: 'root'` / `@Service()`?** A tree-shakable app-wide singleton.

**40. Hierarchical injectors?** Element injectors (component/directive providers) and environment
injectors (root, route, lazy). Lookup walks up; the closest provider wins.

**41. Provider types?** `useClass`, `useValue`, `useFactory`, `useExisting`; `multi: true` for
collections.

**42. InjectionToken?** A token for non-class values (config, interfaces, functions).

**43. `inject()` vs constructor injection?** Equivalent. `inject()` works in functional APIs and
field initializers and is the modern recommendation. It requires an injection context.

**44. Resolution modifiers?** `optional`, `self`, `skipSelf`, `host`.

**45. How do you get one service instance per component?** Add it to that component's `providers`.

**46. What's `injectAsync`?** (v22) Lazy-loads a service's code on first use: code splitting for DI.

## G. Signals

**47. What are signals?** Reactive values that track their consumers and notify on change,
enabling fine-grained updates and zoneless apps.

**48. `signal` / `computed` / `effect`?** Writable state / memoized derived state / side effects.

**49. `linkedSignal`?** Writable state that resets from a source computation.

**50. `resource` / `httpResource`?** Signal-based async loading with `value`, `isLoading`, `error`,
`reload`, and automatic re-fetch and cancellation when params change. Stable in v22.

**51. Signals vs Observables?** Signals hold a current value, are synchronous, and auto-track.
Observables are streams over time with operators and lazy subscription. Bridge with
`toSignal`/`toObservable`.

**52. Common signal pitfalls?** Mutating objects in place; writing signals in `computed`; using
`effect` to derive state; forgetting `()` in templates.

**53. Compare signals to React hooks.** `signal` ≈ `useState`, `computed` ≈ `useMemo`, `effect` ≈
`useEffect`, but there are no dependency arrays or stale closures, and the component doesn't re-run.

## H. RxJS

**54. Observable vs Promise?** Multiple values, lazy, cancellable, operators.

**55. `switchMap` / `mergeMap` / `concatMap` / `exhaustMap`?** Cancel previous / parallel / sequential
queue / ignore while busy.

**56. Subject types?** `Subject`, `BehaviorSubject` (current value), `ReplaySubject` (replays n),
`AsyncSubject` (last on complete).

**57. Hot vs cold?** Shared producer vs one producer per subscriber.

**58. How do you prevent subscription leaks?** `async` pipe, `toSignal`, `takeUntilDestroyed`,
`takeUntil(destroy$)`, unsubscribe.

**59. `forkJoin` vs `combineLatest`?** All complete → one emission vs emit on any change.

**60. Typeahead implementation?** `valueChanges.pipe(debounceTime, distinctUntilChanged,
switchMap(q => api.search(q).pipe(catchError(() => of([])))))`.

**61. What does `shareReplay` do?** Multicasts and caches the last n values so multiple
subscribers don't re-execute the source (e.g. HTTP).

## I. HTTP

**62. How do you set up HttpClient?** `provideHttpClient(withFetch(), withInterceptors([...]))`.

**63. Interceptors?** Functional `HttpInterceptorFn(req, next)`: auth headers, error handling,
logging, caching. Requests are immutable, so clone them.

**64. Why isn't my request sent?** No subscription (observables are lazy).

**65. How do you test HTTP?** `provideHttpClientTesting()` + `HttpTestingController`.

## J. Routing

**66. Lazy loading?** `loadComponent` / `loadChildren` with dynamic imports; preloading strategies.

**67. Guards?** `canActivate`, `canActivateChild`, `canDeactivate`, `canMatch` (functional). They
return boolean/UrlTree/Observable/Promise.

**68. canActivate vs canMatch?** canMatch decides if the route config matches at all (skips the
lazy download, allows alternate routes).

**69. Resolvers?** Fetch data before activation. Trade-off: delayed navigation. Router resources
(v22.2 preview) are the signal-based successor.

**70. How do you read route params?** `input()` with `withComponentInputBinding()`, or
`ActivatedRoute.paramMap` (observable, handles component reuse).

**71. `routerLink` vs `href`?** Client-side navigation without a reload.

## K. Forms

**72. Template-driven vs reactive vs signal forms?** Template-driven: `ngModel`, simple, the model
is in the template. Reactive: explicit typed `FormGroup`s in the class, testable, dynamic. Signal
Forms (stable v22): the model is a signal, schema validation, `[formField]`, the new recommended
API for new code.

**73. FormControl / FormGroup / FormArray?** A single value / named group / dynamic list.

**74. Custom and async validators?** Functions returning `ValidationErrors | null` (or
Observable/Promise). Async validators set `pending`.

**75. ControlValueAccessor?** The bridge for custom inputs: `writeValue`, `registerOnChange`,
`registerOnTouched`, `setDisabledState`.

**76. `setValue` vs `patchValue`?** All fields required vs partial.

## L. Change detection & performance

**77. How does change detection work?** zone.js-triggered top-down binding checks; or zoneless,
with signals/events scheduling checks of affected views.

**78. Default (Eager) vs OnPush?** Eager checks every cycle; OnPush checks on input reference
change, template events, signal changes, async pipe, or markForCheck. OnPush is the default in v22.

**79. Zoneless benefits?** Smaller bundles, no monkey-patching, better performance and stack
traces. It requires signal-driven state.

**80. ExpressionChangedAfterItHasBeenCheckedError?** A value changed after it was checked in the
same cycle (unidirectional flow violated). Dev-only check.

**81. How do you optimize a slow app?** OnPush + signals, `track`, lazy routes, `@defer`, virtual
scroll, pure pipes/computed instead of template method calls, `NgOptimizedImage`, SSR +
hydration, bundle analysis, Angular DevTools profiler.

**82. `markForCheck` vs `detectChanges`?** Schedule a check of this component and its ancestors vs
run CD synchronously on this subtree.

## M. State, architecture, testing, tooling

**83. State management options?** Signals in components/services → NgRx SignalStore → NgRx Store.
Server state: `httpResource` / TanStack Query.

**84. What is NgRx?** A Redux implementation for Angular: actions, reducers, selectors, effects
(RxJS side effects), entity, devtools.

**85. Smart vs dumb components?** Containers inject services and handle data; presentational
components use only inputs/outputs.

**86. NgModule: declarations/imports/exports/providers?** See chapter 16.

**87. How do you test a component?** TestBed (`createComponent`, `setInput`, `whenStable`) or
Angular Testing Library; mock providers with `useValue`.

**88. What's the default test runner now?** Vitest (since v21); Karma is deprecated.

**89. SSR & hydration?** `@angular/ssr`, per-route `RenderMode` (Server/Prerender/Client),
`provideClientHydration(withEventReplay(), withIncrementalHydration())`.

**90. How do you upgrade Angular?** `ng update` one major at a time, following the update guide.
Migrations rewrite code.

## N. Scenario / design questions (senior)

**91. Design a large enterprise Angular app.**
Feature-based folders (or an Nx monorepo with boundaries), standalone + lazy routes per feature,
core/shared split, signal stores per feature (SignalStore or NgRx for complex domains), typed
API layer + interceptors, OnPush/zoneless, design system (Material/Aria/CDK), SSR where SEO
matters, Vitest + Playwright, CI with budgets and lint rules, incremental upgrade policy.

**92. A list of 10,000 rows is slow. What do you do?**
Virtual scroll (CDK), `track` by id, OnPush/signals, paginate or server filter, avoid template
function calls, `@defer` heavy cell content, profile with DevTools.

**93. Migrate an NgModule/zone.js app to modern Angular?**
`ng update` to latest, then run schematics (standalone, control-flow, inject, signal inputs/
outputs/queries), introduce signals in leaf components, enable OnPush, remove zone-dependent
patterns (`setTimeout` mutations), then go zoneless. Do it incrementally, behind tests.

**94. How do you handle auth?**
AuthStore (signals) + login API; token in memory/HTTP-only cookie; `authInterceptor` attaches the
token and handles 401 (refresh/redirect); `authGuard`/`canMatch` for routes; `*appHasRole`
directive for UI; logout clears state.

**95. Coming from React, what was hardest / how do you think about Angular?**
Good answer: the class instance persists (no stale closures), DI replaces Context and prop
drilling, RxJS for streams, signals feel like hooks without dependency arrays, and the framework
provides the conventions React leaves to you.
