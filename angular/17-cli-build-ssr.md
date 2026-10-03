# 17 — CLI, Build, SSR & Deployment

## 1. Angular CLI essentials

| Command | Does | React world |
|---|---|---|
| `ng new app` | Scaffold project (routing, styles, SSR options) | `npm create vite@latest` / `create-next-app` |
| `ng serve` | Dev server with HMR at :4200 | `npm run dev` |
| `ng build` | Production build to `dist/` (default config: production) | `npm run build` |
| `ng test` | Unit tests | `npm test` |
| `ng e2e` | E2E tests (after adding a runner) | — |
| `ng lint` | Lint (after `ng add @angular-eslint/schematics`) | eslint |
| `ng generate` / `ng g` | Generate code (see below) | — (manual) |
| `ng add <pkg>` | Install + configure a library (`@angular/material`, `@angular/ssr`, `@ngrx/store`) | npm install + manual setup |
| `ng update` | Upgrade Angular + run code migrations automatically | manual codemods |
| `ng mcp` | Angular MCP server for AI coding tools (v20.2+) | — |

```bash
ng g component features/users/user-card    # ng g c
ng g service features/users/user-api       # ng g s
ng g directive shared/click-outside        # ng g d
ng g pipe shared/truncate                  # ng g p
ng g guard core/auth                       # functional guard by default
ng g interceptor core/auth
ng g resolver features/users/user
ng g environments
ng g interface models/user
```
Useful flags: `--dry-run`, `--skip-tests`, `--inline-template`, `--inline-style`, `--flat`.

### Updating versions (a common interview + job task)
```bash
ng update                                   # shows what can be updated
ng update @angular/core@22 @angular/cli@22  # one major at a time: 19 → 20 → 21 → 22
```
Check <https://angular.dev/update-guide> for each hop. Migrations rewrite code for you.

## 2. Project structure

```
my-app/
├── angular.json          # workspace/build config (like vite.config + scripts)
├── package.json
├── tsconfig.json / tsconfig.app.json / tsconfig.spec.json
├── public/               # static assets copied as-is (favicon, images)
└── src/
    ├── index.html
    ├── main.ts           # bootstrapApplication
    ├── main.server.ts    # SSR entry (if SSR)
    ├── server.ts         # Express server (if SSR)
    ├── styles.css        # global styles
    └── app/              # app code
```

### `angular.json` things you'll touch
- `architect.build.options`: `styles`, `scripts`, `assets`, `polyfills` (remove `zone.js` for zoneless).
- `configurations.production.budgets`: bundle size limits (warnings/errors).
- `fileReplacements` for environment files.
- `serve.options.proxyConfig`.

## 3. The build system

- **`@angular/build:application`**: esbuild for bundling, Vite for the dev server. The default since
  v17. Fast builds, ES modules, HMR for styles and templates.
- Legacy **webpack** builder (`@angular-devkit/build-angular:browser`) is **deprecated as of v22**;
  migrate with `ng update` → the `use-application-builder` migration.
- **AOT** (ahead-of-time) compilation is always on. Templates compile at build time, so template
  errors are build errors.

## 4. Server-side rendering (SSR), hydration, prerendering

```bash
ng new my-app --ssr        # or add later:
ng add @angular/ssr
```

```ts
// app.config.ts
import { provideClientHydration, withEventReplay, withIncrementalHydration } from '@angular/platform-browser';

providers: [
  provideClientHydration(
    withEventReplay(),            // replays clicks that happen before hydration
    withIncrementalHydration(),   // hydrate @defer blocks on demand
  ),
]
```

```ts
// app.routes.server.ts — per-route render mode
import { RenderMode, ServerRoute } from '@angular/ssr';

export const serverRoutes: ServerRoute[] = [
  { path: '', renderMode: RenderMode.Prerender },             // SSG at build time
  { path: 'products/:id', renderMode: RenderMode.Server },    // SSR per request
  { path: 'dashboard/**', renderMode: RenderMode.Client },    // CSR only
  { path: '**', renderMode: RenderMode.Server },
];
```

Incremental hydration in templates:
```html
@defer (hydrate on viewport) {
  <app-reviews />        <!-- server-rendered HTML, JS loaded/hydrated only when visible -->
}
```

Next.js analogy: `RenderMode.Prerender` ≈ SSG, `Server` ≈ SSR, `Client` ≈ client-only, and
incremental hydration ≈ partial/progressive hydration.

### SSR gotchas
- No `window`/`document`/`localStorage` on the server. Use `afterNextRender`, `isPlatformBrowser(inject(PLATFORM_ID))`, or inject `DOCUMENT`.
- HTTP requests made during SSR are cached and transferred to the client (no double fetch) with
  `provideClientHydration()`.
- Hydration requires the DOM produced by the server to match the client. Avoid direct DOM
  manipulation that changes structure.

## 5. Environments & configuration

```bash
ng g environments
ng build --configuration=staging
```
Many teams prefer **runtime config** (fetch `/config.json` in `provideAppInitializer`) so one build
artifact can be deployed to every environment.

## 6. Deployment

- `ng build` → static files in `dist/<app>/browser`. Host on any static host/CDN (Netlify,
  Vercel, Firebase Hosting, S3 + CloudFront, Nginx).
- **SPA fallback:** configure the server to return `index.html` for unknown routes, otherwise deep links 404.
- With SSR: deploy the Node server (`dist/<app>/server/server.mjs`), or use platform adapters.
- `ng deploy` with builders such as `@angular/fire`.

## 7. The official ecosystem (good to know)

| Package | Purpose |
|---|---|
| `@angular/material` | Material Design components |
| `@angular/cdk` | Unstyled behavior primitives: overlay, drag-drop, virtual scroll, a11y, portal |
| `@angular/aria` | Headless accessible UI patterns (stable in v22) |
| `@angular/animations` | Animation DSL (newer apps can use native CSS with `animate.enter` / `animate.leave`) |
| `@angular/localize` | i18n (`i18n` attributes, `$localize`) |
| `@angular/pwa` | Service worker / PWA |
| `@angular/fire` | Firebase integration |
| Community | NgRx, Nx, Angular Testing Library, Transloco/ngx-translate, PrimeNG, Spartan, TanStack Query |

## 8. Interview questions

1. **What is AOT? JIT?** AOT compiles templates at build time (faster startup, smaller bundles,
   early errors); JIT compiled in the browser at runtime (legacy dev mode).
2. **What does `ng update` do?** Updates packages and runs migration schematics that rewrite your code.
3. **What is hydration?** Reusing server-rendered DOM on the client and attaching listeners instead
   of re-rendering. Incremental hydration delays this per `@defer` block.
4. **SSR vs SSG vs CSR in Angular?** Configured per route via `RenderMode.Server` / `Prerender` / `Client`.
5. **How do you handle browser-only APIs with SSR?** `afterNextRender`, `isPlatformBrowser`, DI
   tokens like `DOCUMENT`, and guarding code paths.
6. **What are bundle budgets?** Size thresholds in `angular.json` that warn or fail builds.
