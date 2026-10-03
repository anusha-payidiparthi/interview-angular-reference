# 11 — Routing

| | AngularJS 1.x | React Router (v6+) | Angular Router |
|---|---|---|---|
| Config | `$routeProvider.when()` / ui-router `$stateProvider.state()` | `<Routes><Route>` or `createBrowserRouter` | `Routes` array + `provideRouter()` |
| Outlet | `ng-view` / `ui-view` | `<Outlet />` | `<router-outlet />` |
| Link | `href="#!/x"` / `ui-sref` | `<Link to>` | `<a routerLink>` |
| Params | `$routeParams` / `$stateParams` | `useParams()` | `input()` binding or `ActivatedRoute` |
| Navigate | `$location.path()` / `$state.go()` | `useNavigate()` | `inject(Router).navigate()` |
| Data before render | `resolve` | `loader` | `resolve` (`ResolveFn`) |
| Protect routes | `$routeChangeStart` hacks / ui-router hooks | wrapper components / loaders | **Guards** (`CanActivateFn`, etc.) |
| Lazy loading | ocLazyLoad (3rd party) | `lazy()` / `React.lazy` | `loadComponent` / `loadChildren` |

## 1. Setup

```ts
// app.routes.ts
import { Routes } from '@angular/router';
import { Home } from './home';
import { NotFound } from './not-found';
import { authGuard } from './core/auth.guard';

export const routes: Routes = [
  { path: '', component: Home, title: 'Home' },                 // title sets document.title
  { path: 'login', loadComponent: () => import('./auth/login').then(m => m.Login) },

  {
    path: 'users',
    canActivate: [authGuard],
    loadComponent: () => import('./users/users-layout').then(m => m.UsersLayout),
    children: [                                                  // nested routes
      { path: '', loadComponent: () => import('./users/user-list').then(m => m.UserList) },
      {
        path: ':id',
        loadComponent: () => import('./users/user-detail').then(m => m.UserDetail),
        resolve: { user: userResolver },
      },
    ],
  },

  // lazy-load a whole feature's route file
  { path: 'admin', loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES) },

  { path: 'old-path', redirectTo: 'users', pathMatch: 'full' },
  { path: '**', component: NotFound },                           // wildcard — keep LAST
];
```

```ts
// app.config.ts
import { provideRouter, withComponentInputBinding, withViewTransitions, withInMemoryScrolling } from '@angular/router';

providers: [
  provideRouter(
    routes,
    withComponentInputBinding(),          // route params/query/data → component inputs
    withViewTransitions(),                // optional: animated page transitions
    withInMemoryScrolling({ scrollPositionRestoration: 'enabled' }),
  ),
]
```

```ts
// app.ts
import { RouterOutlet, RouterLink, RouterLinkActive } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <nav>
      <a routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">Home</a>
      <a routerLink="/users" routerLinkActive="active">Users</a>
      <a [routerLink]="['/users', 42]" [queryParams]="{ tab: 'orders' }">User 42 orders</a>
    </nav>
    <router-outlet />
  `,
})
export class App {}
```
Child routes render in a `<router-outlet />` inside the parent's template (`UsersLayout`).

### `pathMatch`
- `'prefix'` (default): matches if the URL *starts with* the path.
- `'full'`: the whole remaining URL must match. **Required for `redirectTo` on empty path**, or
  every URL would redirect.

## 2. Reading route params

### Modern: params as inputs (`withComponentInputBinding()`)
```ts
// route: { path: 'users/:id', ... }   URL: /users/42?tab=orders
export class UserDetail {
  id = input.required<string>();      // from :id  (always string — convert if needed)
  tab = input<string>();              // from ?tab=
  user = input<User>();               // from resolve: { user: ... } or data: { user: ... }

  userId = computed(() => Number(this.id()));
}
```

### Classic: `ActivatedRoute`
```ts
export class UserDetail {
  private route = inject(ActivatedRoute);

  // snapshot: one-time read (breaks if navigating /users/1 → /users/2 reuses the component!)
  idOnce = this.route.snapshot.paramMap.get('id');

  // observable: updates on param changes — the safe way
  user = toSignal(
    this.route.paramMap.pipe(
      map(p => Number(p.get('id'))),
      switchMap(id => inject(UserApi).get(id)),   // ❌ inject() here is NOT an injection context
    ),
  );
}
```
Fixed version: capture the service first.
```ts
private api = inject(UserApi);
user = toSignal(this.route.paramMap.pipe(
  map(p => Number(p.get('id'))),
  switchMap(id => this.api.get(id)),
));
```
Also available: `route.queryParamMap`, `route.data`, `route.parent`, `route.fragment`.

> **Gotcha:** Angular **reuses** the component when only params change (`/users/1` → `/users/2`).
> `ngOnInit` won't run again, so use observables or input signals, not the snapshot.

## 3. Navigating in code

```ts
private router = inject(Router);

this.router.navigate(['/users', id]);
this.router.navigate(['../edit'], { relativeTo: this.route });   // relative
this.router.navigate([], { queryParams: { page: 2 }, queryParamsHandling: 'merge' });
this.router.navigateByUrl('/login?returnUrl=%2Fusers');
```

## 4. Guards (functional, v15+)

| Guard | Purpose |
|---|---|
| `canActivate` | May this route be entered? |
| `canActivateChild` | May child routes be entered? |
| `canDeactivate` | May the user leave? (unsaved changes) |
| `canMatch` | Should this route config even match? (feature flags, role-based alternate routes; also prevents lazy chunk download) |

```ts
// auth.guard.ts
import { CanActivateFn, Router } from '@angular/router';

export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthStore);
  const router = inject(Router);
  return auth.isLoggedIn()
    ? true
    : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });  // redirect
};

// role guard factory
export const roleGuard = (role: string): CanMatchFn => () => inject(AuthStore).hasRole(role);

// routes
{ path: 'admin', canMatch: [roleGuard('admin')], loadChildren: ... }
```

Guards can return `boolean | UrlTree | Observable<...> | Promise<...>`. Returning a `UrlTree`
redirects (prefer it over calling `router.navigate` inside a guard).

```ts
// unsaved-changes.guard.ts
export interface HasUnsavedChanges { hasUnsavedChanges(): boolean }

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = component =>
  component.hasUnsavedChanges() ? confirm('Discard changes?') : true;
```

React comparison: in React Router you'd wrap routes in `<RequireAuth>` or check in a `loader`
and `redirect()`. Angular guards are the same idea, declared in route config.

Legacy: class guards `implements CanActivate` with `canActivate()` method (deprecated style).

## 5. Resolvers — fetch before navigating

```ts
export const userResolver: ResolveFn<User> = route =>
  inject(UserApi).get(Number(route.paramMap.get('id')));

// route: { path: ':id', component: UserDetail, resolve: { user: userResolver } }
// component: user = input.required<User>();   (with withComponentInputBinding)
```
Trade-off: resolvers delay navigation until data arrives (no spinner inside the page, but the
page feels stuck on slow APIs). Many teams prefer loading inside the component with skeletons.

React Router equivalent: `loader` + `useLoaderData()`.

### Router resources (v22.2, developer preview)
The signal-based successor to resolvers: routes declare **resources** that load data before or
during navigation. Resources for all matched routes (parent and child) load **in parallel**,
avoiding the parent→child waterfall that nested resolvers cause, and they expose the usual resource
state (`value`, `isLoading`, `error`). The API is still in preview, so check angular.dev → Routing
before adopting it. For now, resolvers or loading in the component with `httpResource` are the
production choices.

Since v22.2, guards, resolvers and resources can also **throw a `RedirectCommand`** to redirect.

## 6. Route data & titles

```ts
{ path: 'reports', component: Reports, data: { breadcrumb: 'Reports', permissions: ['read'] }, title: 'Reports' }
```
Custom title strategy: extend `TitleStrategy` to add a suffix like "| MyApp".

## 7. Lazy loading & preloading

```ts
// standalone component
{ path: 'settings', loadComponent: () => import('./settings/settings').then(m => m.Settings) }

// with a default export you can drop .then():
{ path: 'settings', loadComponent: () => import('./settings/settings') }

// group of routes
{ path: 'shop', loadChildren: () => import('./shop/shop.routes') }

// legacy NgModule
{ path: 'shop', loadChildren: () => import('./shop/shop.module').then(m => m.ShopModule) }
```
Preloading strategy (download lazy chunks after initial load):
```ts
provideRouter(routes, withPreloading(PreloadAllModules))
```

## 7b. Newer router options (v21–v22)

```ts
provideRouter(
  routes,
  withComponentInputBinding(),
  withAutoCleanupInjectors(),               // destroy route-level providers when routes become inactive (stable v22.2)
  withExperimentalPlatformNavigation(),     // use the browser's native Navigation API (experimental, v22)
)
```

## 8. Router events

```ts
inject(Router).events.pipe(
  filter((e): e is NavigationEnd => e instanceof NavigationEnd),
  takeUntilDestroyed(),
).subscribe(e => analytics.pageView(e.urlAfterRedirects));
```
Events: `NavigationStart`, `RoutesRecognized`, `GuardsCheckStart`, `ResolveStart`,
`NavigationEnd`, `NavigationCancel`, `NavigationError`.

## 9. Interview questions

1. **How does lazy loading work?** `loadComponent`/`loadChildren` with dynamic `import()` produce
   separate chunks fetched on first navigation. Preloading can fetch them in the background.
2. **Types of guards?** canActivate, canActivateChild, canDeactivate, canMatch (plus legacy
   canLoad, replaced by canMatch).
3. **canActivate vs canMatch?** canActivate runs after the route matched (the lazy chunk may
   already be downloaded); canMatch decides whether the route matches at all, so it can skip to
   another route config and avoid downloading the chunk.
4. **What's a resolver? Pros/cons?** Pre-fetches data before activation. It guarantees data is
   present but delays navigation.
5. **snapshot vs observable params?** The snapshot is a one-time value. Observables update when the
   component is reused for new params.
6. **`pathMatch: 'full'` — why?** An empty-path redirect with prefix matching would match every URL.
7. **How do you pass route params to components without ActivatedRoute?** `withComponentInputBinding()`
   + `input()`s named like the params.
8. **`routerLink` vs `href`?** `routerLink` navigates client-side without a full page reload.
