# 16 — NgModules (Legacy) & Application Architecture

New code is standalone, but **most company codebases started before v15–v17** and still contain
NgModules. You must be able to read them, work in them, and migrate them.

## 1. NgModule anatomy

```ts
@NgModule({
  declarations: [UserList, UserCard, TruncatePipe],  // components/directives/pipes OWNED by this module (non-standalone)
  imports: [CommonModule, ReactiveFormsModule, SharedModule, RouterModule.forChild(routes)], // what my templates need
  exports: [UserCard],                               // what modules importing me can use
  providers: [UserService],                          // services (prefer providedIn: 'root' instead)
})
export class UsersModule {}

// root module
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule, AppRoutingModule, CoreModule],
  bootstrap: [AppComponent],
})
export class AppModule {}

// main.ts
platformBrowserDynamic().bootstrapModule(AppModule);
```

AngularJS analogy: `angular.module('users', ['shared'])` grouped things too, but everything was
global once loaded. NgModules introduced **compilation scope**: a component can only use what its
module imports.

Mental model: an NgModule is like a "barrel + DI config + template scope" in one class.

## 2. Classic module types

| Module | Contents | Rule |
|---|---|---|
| `AppModule` | Root component, app-wide setup | Imported once |
| `CoreModule` | Singleton services, interceptors, header/footer | Imported **only** in AppModule (often guarded against re-import) |
| `SharedModule` | Reusable dumb components, pipes, directives; re-exports `CommonModule`, `FormsModule` | Imported by feature modules; **no providers** |
| Feature modules | One per business domain (`UsersModule`, `OrdersModule`) | Usually lazy-loaded |
| Routing modules | `RouterModule.forRoot/forChild(routes)` | `forRoot` once, `forChild` in features |

### `forRoot()` / `forChild()` pattern
```ts
@NgModule({})
export class AnalyticsModule {
  static forRoot(config: AnalyticsConfig): ModuleWithProviders<AnalyticsModule> {
    return { ngModule: AnalyticsModule, providers: [{ provide: ANALYTICS_CONFIG, useValue: config }] };
  }
}
```
It exists to avoid duplicate service instances in lazy-loaded modules. The standalone equivalent is a
`provideAnalytics(config)` function returning `EnvironmentProviders`, the same style as
`provideRouter`/`provideHttpClient`:
```ts
export function provideAnalytics(config: AnalyticsConfig): EnvironmentProviders {
  return makeEnvironmentProviders([{ provide: ANALYTICS_CONFIG, useValue: config }, AnalyticsService]);
}
```

## 3. Mixing standalone and NgModules

```ts
// standalone component using an NgModule-based library
@Component({ imports: [MatButtonModule, SomeLegacyModule], ... })

// NgModule using a standalone component: import it (don't declare it)
@NgModule({ imports: [UserCard], declarations: [LegacyPage] })
```

## 4. Migrating to standalone

```bash
ng generate @angular/core:standalone        # 1) convert components, 2) remove modules, 3) bootstrap
ng generate @angular/core:control-flow      # *ngIf → @if
ng generate @angular/core:signal-input-migration
ng generate @angular/core:output-migration
ng generate @angular/core:signal-queries-migration
ng generate @angular/core:inject            # constructor → inject()
ng generate @angular/core:route-lazy-loading
```
Real-world migrations are incremental: new features are standalone, and old modules are converted
when touched.

## 5. Recommended folder structure (standalone era)

```
src/app/
├── app.ts / app.html / app.config.ts / app.routes.ts
├── core/                    # app-wide singletons: auth, interceptors, guards, layout shell
│   ├── auth/
│   │   ├── auth.store.ts
│   │   ├── auth.guard.ts
│   │   └── auth.interceptor.ts
│   └── layout/header.ts
├── shared/                  # reusable, dumb, framework-level UI and utils
│   ├── ui/button.ts, card.ts, spinner.ts
│   ├── pipes/truncate.pipe.ts
│   └── directives/click-outside.ts
└── features/                # one folder per domain, lazy loaded
    ├── users/
    │   ├── users.routes.ts
    │   ├── data/user-api.ts, user.store.ts, user.model.ts
    │   ├── pages/user-list-page.ts, user-detail-page.ts       # smart (container)
    │   └── ui/user-card.ts, user-form.ts                       # dumb (presentational)
    └── orders/...
```
Feature-based (not type-based) organization, the same advice as in React. Large monorepos
often use **Nx** with enforced module boundaries.

## 6. Smart vs presentational components

| Smart / container (page) | Presentational / dumb (ui) |
|---|---|
| Injects services/stores | No service injection (ideally) |
| Fetches data, handles routing | Only `input()` / `output()` |
| Little markup | Most markup and styling |
| Hard to reuse | Reusable, easy to test, OnPush-friendly |

Identical to React's container/presentational split.

## 7. Style guide highlights (angular.dev/style-guide)

- One concept per file; files named in kebab-case: `user-profile.ts` (v20+), or the older
  `user-profile.component.ts`.
- Prefer `inject()` over constructor injection.
- Use `protected` for template-only members, and `readonly` for inputs, outputs and queries.
- Keep components focused on presentation; move logic into services.
- Prefer `[class.x]`/`[style.x]` bindings over `ngClass`/`ngStyle`.
- Group Angular-specific properties (inputs, outputs, injected deps) before methods.

## 8. Interview questions

1. **What is an NgModule?** A class decorated with `@NgModule` that declares components, imports
   dependencies for their templates, exports public pieces, and configures providers.
2. **declarations vs imports vs exports vs providers?** Owned non-standalone declarables; other
   modules/standalone pieces my templates need; what I share; services for DI.
3. **Why standalone?** Less boilerplate, clearer dependencies per component, better tree-shaking
   and lazy loading, easier learning curve.
4. **What is `forRoot`?** A static method returning the module with providers, imported once at the
   root to avoid duplicate singletons; `forChild` omits them.
5. **SharedModule vs CoreModule?** Shared is reusable declarables imported everywhere with no
   providers. Core is singletons imported once.
6. **How would you migrate a large NgModule app to standalone?** Use the schematics incrementally,
   start from leaves (shared components), then features, then bootstrap. Convert lazy modules to
   `loadChildren` routes files.
