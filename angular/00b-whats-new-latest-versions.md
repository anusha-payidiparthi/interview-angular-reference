# 00b — What's New: Angular v21 & v22 (Latest)

> **Current status (Oct 2026):** Angular **v22** is the active major (released June 3, 2026;
> latest minor **v22.2**, v22.3 due ~Nov 2026). **v21** and **v20** are in LTS (security fixes
> only). Starting with v22, Angular moved to **one major per year** (v23 ≈ June 2027) with 4–6
> minors per major.
> Always confirm on <https://angular.dev/reference/releases>.

---

## 1. What version will I see at a company?

| You'll find | How common | What it looks like |
|---|---|---|
| **v20–v22** (modern) | New projects, well-maintained apps | Standalone, signals, `@if/@for`, `input()`, zoneless/OnPush, Vitest |
| **v16–v19** (transitional) | **Very common** | Mix: standalone + some NgModules, `@Input()` + some signals, `*ngIf` and `@if` side by side, zone.js, Karma/Jasmine or Jest |
| **v8–v15** (legacy) | Enterprises with upgrade backlog | NgModules everywhere, RxJS for all state, `*ngIf`, constructor injection, class guards |

**Strategy:** learn the **v22 way** (this guide's default) and be fluent at **reading** the
older syntax (each chapter has a "legacy" section). Companies upgrade one major at a time, and
upgrade work is a common task for new hires.

---

## 2. Angular v22 (June 2026) — headline features

### Now stable (production-ready)
| Feature | What it is | Chapter |
|---|---|---|
| **Signal Forms** | New forms API: model is a signal, schema-based validation, `[formField]` binding | [12](./12-forms.md) |
| **`resource` / `httpResource`** | Async data as signals (`value()`, `isLoading()`, `error()`, `reload()`). Comparable to React Query built in | [08](./08-signals.md), [10](./10-http-client.md) |
| **Angular Aria** (`@angular/aria`) | Headless, accessible UI patterns (tree, tabs, menu, combobox, listbox…), with test harnesses | [17](./17-cli-build-ssr.md) |

### Change detection: `OnPush` is the default
```ts
@Component({ selector: 'app-weather', template: `...` })   // ← OnPush by default in v22
export class Weather {}

// Old default renamed: ChangeDetectionStrategy.Default → ChangeDetectionStrategy.Eager
@Component({ ..., changeDetection: ChangeDetectionStrategy.Eager })
```
Combined with zoneless (default since v21), new apps are "performance by default". When you
`ng update` an existing app, the migration keeps old behavior explicit (`Eager`) where needed.
See [chapter 13](./13-change-detection-and-performance.md).

### New `@Service()` decorator
```ts
import { Service } from '@angular/core';

@Service()                        // = @Injectable({ providedIn: 'root' }) for the common case
export class CartStore { ... }
```
`@Injectable` still exists for advanced configuration. See [chapter 07](./07-dependency-injection.md).

### `injectAsync` — lazy-loaded services (code splitting for DI)
```ts
import { Component, injectAsync } from '@angular/core';

@Component({ selector: 'app-report', template: `<button (click)="export()">Export</button>` })
export class Report {
  private exporter = injectAsync(() => import('./report-exporter'));   // service chunk not loaded yet

  async export() {
    const exporter = await this.exporter();   // downloaded on first use
    exporter.export();
  }
}
```
The service must be auto-provided (e.g. with `@Service()`). It supports a `prefetch` option (e.g. on idle).

### Template upgrades
```html
<!-- Arrow functions in templates -->
<button (click)="item.update(p => ({ ...p, stock: p.stock - 1 }))">-1</button>

<!-- Spread / rest in object & array literals and calls -->
<div [class]="{ ...baseClasses, active: isActive() }"></div>
<app-cart [items]="[...defaults, 'croissant']" />

<!-- @switch: multiple cases share a block -->
@switch (status()) {
  @case ('pending')
  @case ('processing') { <p>In progress</p> }
  @case ('shipped') { <p>On its way</p> }
  @default never;          <!-- exhaustive check: compile error if a union member is unhandled -->
}

<!-- Comments inside element tags -->
<input
  // the user's email
  type="email"
  /* bound to signal form */
  [formField]="f.email" />
```
See [chapter 03](./03-data-binding-and-control-flow.md).

### Router
- **Native Navigation API integration** (experimental): `provideRouter(routes, withExperimentalPlatformNavigation())`
  gives native scroll restoration and navigation lifecycle hooks.
- **Route injector cleanup**: `withAutoCleanupInjectors()` (stable in v22.2) destroys providers of inactive routes.
- `destroyDetachedRouteHandle` for custom `RouteReuseStrategy`.

### Removals and deprecations
- `ComponentFactoryResolver` / `ComponentFactory` **removed**. Pass the class to `ViewContainerRef.createComponent(Cmp)`.
- **Webpack builders deprecated** (`@angular-devkit/build-angular`, `@ngtools/webpack`). Use the esbuild `application` builder.
- TypeScript 6 support.

### AI tooling
- Angular **MCP server** (`ng mcp`) tools for agents: dev server control, `modernize`,
  `onpush_zoneless_migration`, `ai_tutor`.
- **Angular Agent Skills** (`angular-developer`, `angular-new-app`).
- Experimental **WebMCP** support: expose app features as tools to in-browser AI agents.

---

## 3. Angular v22.1 / v22.2 (Jul–Sep 2026)

| Feature | Status | What it is |
|---|---|---|
| **`@boundary`** | Developer preview | Template **error boundaries**, like React Error Boundaries |
| **Router resources** | Developer preview | Signal/resource-based data loading for routes (resolver successor). Loads in parallel across matched routes and avoids waterfalls |
| **`private` members in templates** | Stable | Templates can now access `private` class members (previously `protected` was the minimum) |
| `withAutoCleanupInjectors` | Stable | (was experimental) |
| `RedirectCommand` can be thrown from guards/resolvers/resources | Stable | Simpler redirects |
| `strictUnclaimedEventNames` | Compiler flag | Catches typos in custom event bindings |

```html
@boundary {
  <app-promo-widget />          <!-- if this throws, the rest of the page survives -->
} @error (let err) {
  <app-default-promo />
}
```
React analogy: `<ErrorBoundary fallback={<DefaultPromo/>}><PromoWidget/></ErrorBoundary>`.

---

## 4. Angular v21 (Nov 2025) — what it introduced

| Feature | Notes |
|---|---|
| **Zoneless by default** for new apps | No `zone.js`; CD driven by signals/events (`provideZonelessChangeDetection` stable since v20.2) |
| **Vitest** default test runner | Karma deprecated; Jasmine still supported |
| **Signal Forms** (experimental) | Became stable in v22 |
| **Angular Aria** (developer preview) | Became stable in v22 |
| `HttpClient` provided by default | You still call `provideHttpClient(...)` to add interceptors/options |
| MCP server tools expanded | `ng mcp` |

## 5. Angular v20 (May 2025) — still in LTS, very common

- `effect`, `linkedSignal`, `toSignal`, signal inputs/outputs/queries **stable**.
- Zoneless **stable** (v20.2).
- `httpResource` introduced.
- `afterRender` renamed `afterEveryRender`.
- New **style guide**: no `.component` / `Component` suffixes for new files (`user-card.ts` → `class UserCard`).
- Template: exponentiation `**`, `in` operator, `void`, template literals.
- Animations: native `animate.enter` / `animate.leave` (v20.2), with `@angular/animations` on the way out.

---

## 6. The "modern Angular" checklist (what a v22 codebase looks like)

- [ ] Standalone components only (no `standalone: true` needed, it's the default)
- [ ] `bootstrapApplication` + `app.config.ts` with `provide*()` functions
- [ ] Zoneless, OnPush default
- [ ] `signal` / `computed` / `linkedSignal` for state; `effect` only for side effects
- [ ] `input()` / `output()` / `model()` / `viewChild()` instead of decorators
- [ ] `@if` / `@for` / `@switch` / `@let` / `@defer` instead of `*ngIf` etc.
- [ ] `inject()` instead of constructor injection; `@Service()` for root singletons
- [ ] `httpResource` / `resource` for reads; `HttpClient` for writes
- [ ] Signal Forms for new forms (Reactive Forms still fine and widespread)
- [ ] Functional guards, resolvers (or router resources), interceptors
- [ ] Lazy routes with `loadComponent` / `loadChildren`
- [ ] Vitest, Angular Testing Library or TestBed with `whenStable()`
- [ ] esbuild application builder, SSR + incremental hydration where it matters

## 7. Interview questions on recent versions

1. **What changed in Angular recently?** Signals everywhere (inputs, queries, forms, resources),
   zoneless + OnPush by default, built-in control flow, standalone by default, `@defer` and
   incremental hydration, and stable Signal Forms and `httpResource` in v22.
2. **Why did Angular make OnPush the default?** With signals and zoneless, Angular knows exactly what
   changed, so checking only affected components is safe and faster ("performance by default").
3. **What is `ChangeDetectionStrategy.Eager`?** The renamed old `Default` strategy, which checks the
   component on every change detection run.
4. **What is `@Service()`?** A v22 decorator for root-provided singleton services, shorthand for
   `@Injectable({ providedIn: 'root' })`.
5. **How would you lazy-load a heavy service?** `injectAsync(() => import('./heavy-service'))` (v22).
6. **Does Angular have error boundaries like React?** Yes: `@boundary { } @error { }` (developer
   preview in v22.2).
7. **What's the release cadence now?** One major per year since v22, with 4–6 minors. v22 is
   supported for 24 months in total: active until ~June 2027, then LTS until ~June 2028. Before
   v22 it was a major every 6 months (check the releases page for exact dates).
