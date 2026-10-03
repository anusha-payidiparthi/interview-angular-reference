# 18 — Cheat Sheet: AngularJS 1.x → Angular (v22) → React

One lookup table for everything. "Legacy Angular" is the pre-signals syntax you'll still see in
company codebases.

## Building blocks

| Concept | AngularJS 1.x | Angular (modern, v22) | Legacy Angular (v2–v16 style) | React |
|---|---|---|---|---|
| App bootstrap | `ng-app` / `angular.bootstrap` | `bootstrapApplication(App, appConfig)` | `platformBrowserDynamic().bootstrapModule(AppModule)` | `createRoot().render(<App/>)` |
| Grouping | `angular.module()` | ES modules + standalone `imports` | `@NgModule` | ES modules |
| Component | `.component({ bindings, controller, template })` | `@Component({ selector, imports, template })` class | same, `standalone: false` + declared in NgModule | function component |
| Controller | `ng-controller` + `$scope` | component class | component class | component function body |
| Local state | `$scope.x` / `this.x` | `x = signal(0)` | plain property `x = 0` | `useState` |
| Derived state | `$watch` / function in template | `computed(() => ...)` | getter / pipe / RxJS `map` | `useMemo` |
| Side effect on change | `$watch(fn)` | `effect(() => ...)` | `ngOnChanges` / RxJS subscribe | `useEffect` |
| Input prop | `bindings: { x: '<' }` | `x = input<T>()` / `input.required<T>()` | `@Input() x` | props |
| Output/callback | `bindings: { onX: '&' }` | `x = output<T>()` | `@Output() x = new EventEmitter<T>()` | callback prop |
| Two-way | `bindings: { x: '=' }` | `x = model<T>()` | `@Input() x` + `@Output() xChange` | `value` + `onChange` |
| Children/slots | `transclude` + `ng-transclude` | `<ng-content select="...">` | same | `children` / named props |
| DOM ref | `$element` in link fn | `viewChild<ElementRef>('ref')` | `@ViewChild('ref')` | `useRef` |
| Projected child ref | `require` | `contentChild(X)` / `contentChildren(X)` | `@ContentChild` / `@ContentChildren` | `React.Children` |
| Fragment | — | `<ng-container>` | same | `<>...</>` |
| Lazy template chunk | — | `<ng-template>` | same | render prop / function |
| Error boundary | — | `@boundary { } @error { }` (v22.2 preview) | `ErrorHandler` (global only) | `<ErrorBoundary>` |

## Templates

| Concept | AngularJS 1.x | Angular (modern) | Legacy Angular | React (JSX) |
|---|---|---|---|---|
| Text | `{{ x }}` | `{{ x() }}` (signal) | `{{ x }}` | `{x}` |
| Property | `ng-src="{{u}}"`, `ng-disabled="d"` | `[src]="u()"`, `[disabled]="d()"` | same with plain props | `src={u}` |
| Attribute | `ng-attr-x` | `[attr.aria-label]="l()"` | same | `aria-label={l}` |
| Event | `ng-click="f()"` | `(click)="f()"` | same | `onClick={f}` |
| Key event | `ng-keyup` + check | `(keyup.enter)="f()"` | same | `onKeyUp={e => e.key==='Enter' && f()}` |
| Two-way input | `ng-model="x"` | `[(ngModel)]="x"` / `[formField]="f.x"` (Signal Forms) | `[(ngModel)]` / `formControlName` | `value={x} onChange={...}` |
| Condition | `ng-if="c"` | `@if (c) {} @else {}` | `*ngIf="c; else tpl"` | `{c ? <A/> : <B/>}` |
| Loop | `ng-repeat="i in items track by i.id"` | `@for (i of items(); track i.id) {} @empty {}` | `*ngFor="let i of items; trackBy: fn"` | `items.map(i => <X key={i.id}/>)` |
| Switch | `ng-switch` | `@switch (v) { @case ('a') {} @default {} }` | `[ngSwitch]` + `*ngSwitchCase` | `switch` / object map |
| Local var | `ng-init` | `@let x = ...;` | `*ngIf="obs$ \| async as x"` | `const x = ...` |
| Show/hide | `ng-show` / `ng-hide` | `[hidden]="!c"` | same | `style={{display}}` |
| Class | `ng-class="{a: c}"` | `[class.a]="c()"` / `[class]="{a: c()}"` | `[ngClass]` | `className={clsx({a: c})}` |
| Style | `ng-style` | `[style.width.px]="w()"` | `[ngStyle]` | `style={{width: w}}` |
| Formatting | filters `{{ d \| date }}` | pipes `{{ d \| date }}` | same | `{format(d)}` |
| Inline fn | `ng-click="x = x + 1"` | arrow functions allowed (v22): `(click)="s.update(v => v + 1)"` | expressions only | arrow functions |
| Lazy section | — | `@defer (on viewport) {}` | — | `lazy()` + `Suspense` |
| Element ref | — | `#ref` | same | `ref={r}` |

## Logic & services

| Concept | AngularJS 1.x | Angular (modern) | Legacy Angular | React |
|---|---|---|---|---|
| Service definition | `.service()` / `.factory()` | `@Service()` (v22) / `@Injectable({ providedIn: 'root' })` | `@Injectable()` + module `providers` | module / custom hook |
| Get a dependency | `function($http)` | `inject(HttpClient)` | `constructor(private http: HttpClient)` | `useContext` / import |
| Scoped instance | — | component `providers: [X]` | same | Context Provider |
| Config value | `.constant()` / `.value()` | `InjectionToken` + `useValue` | same | Context / env vars |
| Lazy service | — | `injectAsync(() => import(...))` (v22) | — | dynamic `import()` |
| HTTP | `$http.get().then()` | `httpResource(() => url)` / `HttpClient.get()` (Observable) | `HttpClient` + `HttpClientModule` | `fetch` / axios / React Query |
| Interceptor | `$httpProvider.interceptors` | `HttpInterceptorFn` + `withInterceptors` | class `HttpInterceptor` + `HTTP_INTERCEPTORS` | axios interceptors |
| Promise/async | `$q` | Observables / signals / async-await | Observables | Promises |
| Event bus | `$rootScope.$broadcast/$on` | service with signal / `Subject` | `Subject` | Context / emitter |
| Async state | — | `resource()` | — | React Query / `use()` |

## Lifecycle

| AngularJS 1.x | Angular (modern) | Legacy Angular | React |
|---|---|---|---|
| controller fn | constructor (+ `inject`) | constructor | function body |
| `$onInit` | `ngOnInit` (still used) | `ngOnInit` | `useEffect(fn, [])` |
| `$onChanges` | `computed` / `effect` on inputs | `ngOnChanges` | `useEffect(fn, [prop])` |
| `$postLink` | `afterNextRender()` | `ngAfterViewInit` | `useLayoutEffect(fn, [])` |
| `$onDestroy` | `DestroyRef.onDestroy` / `takeUntilDestroyed` | `ngOnDestroy` | effect cleanup |
| `$doCheck` | (rarely needed) | `ngDoCheck` | — |

## Routing

| Concept | AngularJS (ngRoute / ui-router) | Angular | React Router |
|---|---|---|---|
| Define | `$routeProvider.when('/u/:id', {...})` | `{ path: 'u/:id', loadComponent: ... }` | `{ path: 'u/:id', element / lazy }` |
| Outlet | `ng-view` / `ui-view` | `<router-outlet />` | `<Outlet />` |
| Link | `href="#!/u/1"` / `ui-sref` | `routerLink="/u/1"` | `<Link to>` |
| Active link | — | `routerLinkActive="active"` | `<NavLink>` |
| Params | `$routeParams.id` | `id = input()` (with `withComponentInputBinding`) / `ActivatedRoute` | `useParams()` |
| Navigate | `$location.path()` / `$state.go()` | `router.navigate(['/u', 1])` | `navigate('/u/1')` |
| Pre-fetch | `resolve: {}` | `resolve` (`ResolveFn`) / router resources (v22.2) | `loader` |
| Protect | event hooks | `canActivate` / `canMatch` / `canDeactivate` | wrapper / loader redirect |
| Lazy | ocLazyLoad | `loadComponent` / `loadChildren` | `lazy` |

## Forms

| AngularJS | Angular Signal Forms (v22 stable) | Angular Reactive (very common) | Angular Template-driven | React |
|---|---|---|---|---|
| `ng-model` | `form(signal(model), schema)` + `[formField]` | `FormGroup` / `FormControl` + `formControlName` | `[(ngModel)]` | controlled inputs / RHF |
| `required` attr + `form.x.$error` | `required(path.x)` + `f.x().errors()` | `Validators.required` + `ctrl.errors` | `required` attr + `#x="ngModel"` | RHF rules / zod |
| `$valid`, `$touched`, `$dirty` | `f().invalid()`, `touched()`, `dirty()` | `.valid`, `.touched`, `.dirty` | same via `ngModel` | `formState` |
| custom directive validator | `validate(path.x, fn)` | `ValidatorFn` | directive + `NG_VALIDATORS` | custom rule |
| `ngModelController` | `FormValueControl` interface | `ControlValueAccessor` | `ControlValueAccessor` | `Controller` |

## Change detection

| AngularJS | Angular (v22) | Legacy Angular | React |
|---|---|---|---|
| Digest cycle, dirty-check all watchers | Zoneless + OnPush default; signals schedule targeted updates | zone.js triggers top-down check; `Default` (now `Eager`) or `OnPush` | `setState` → re-render subtree → VDOM diff |
| `$scope.$apply()` | `signal.set()` (automatic) | `ChangeDetectorRef.markForCheck()` / `NgZone.run()` | `setState` |
| one-time binding `::x` | signals/OnPush make it unnecessary | `OnPush` | `memo` |

## Tooling

| | AngularJS | Angular | React |
|---|---|---|---|
| Scaffold | yeoman / manual | `ng new` | Vite / Next |
| Build | Grunt / Gulp / webpack | `ng build` (esbuild + Vite) | Vite / webpack / Turbopack |
| Unit tests | Karma + Jasmine | Vitest (default) / Jasmine | Vitest / Jest |
| E2E | Protractor | Playwright / Cypress | Playwright / Cypress |
| Upgrade | manual | `ng update` with migrations | manual / codemods |
| DevTools | Batarang | Angular DevTools | React DevTools |
