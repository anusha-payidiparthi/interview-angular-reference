# Modern Angular (Latest: v22) — From React + AngularJS 1.x to Expert

A self-paced guide for a **React developer** who used **AngularJS 1.x** years ago and now needs to
get productive (and interview-ready) in **modern Angular**: the latest version companies use
(**v22**, released June 2026; v22.2 current as of Oct 2026).

> **Naming:** "Angular 2" (2016) was the ground-up rewrite of AngularJS. Since then the framework
> is just called **Angular**, with versions v2 → v4 → … → v20, v21, **v22**. Everything here is
> written in the **latest style** (standalone components, signals, `@if`/`@for`, zoneless + OnPush,
> Signal Forms, `httpResource`). Each chapter also shows the **legacy syntax** (`NgModule`,
> `@Input()`, `*ngIf`, RxJS-only state) that most company codebases still contain.

---

## How to use this guide

### Phase 1 — The big picture (1 day)
1. **[00 — High-level overview](./00-high-level-overview.md)**: every major feature, what it
   replaces from AngularJS, and what it maps to in React.
2. **[00b — What's new in v21 & v22](./00b-whats-new-latest-versions.md)**: the latest features,
   which versions companies actually run, and the "modern Angular" checklist.

Don't go deep yet.

### Phase 2 — Deep dives with examples (1–2 weeks)
Work through the chapters in order. Each one follows the same layout:
**Concept → AngularJS 1.x way → React way → Angular way → legacy syntax → gotchas → interview Qs.**

| # | Chapter | React analogy |
|---|---------|---------------|
| 01 | [TypeScript essentials](./01-typescript-essentials.md) | React + TS |
| 02 | [Components & templates](./02-components-and-templates.md) | Function components + JSX |
| 03 | [Data binding & control flow](./03-data-binding-and-control-flow.md) | JSX expressions, `&&`, `.map()` |
| 04 | [Component communication](./04-component-communication.md) | props, callbacks, lifting state |
| 05 | [Lifecycle hooks](./05-lifecycle-hooks.md) | `useEffect`, `useLayoutEffect` |
| 06 | [Directives & pipes](./06-directives-and-pipes.md) | custom hooks, HOCs, formatters |
| 07 | [Dependency injection & services](./07-dependency-injection.md) | Context + custom hooks |
| 08 | [Signals](./08-signals.md) | `useState`, `useMemo`, `useEffect` |
| 09 | [RxJS & Observables](./09-rxjs-and-observables.md) | (no direct equivalent) |
| 10 | [HTTP client](./10-http-client.md) | fetch/axios, React Query |
| 11 | [Routing](./11-routing.md) | React Router |
| 12 | [Forms](./12-forms.md) | controlled inputs, React Hook Form |
| 13 | [Change detection & performance](./13-change-detection-and-performance.md) | reconciliation, `memo` |
| 14 | [State management](./14-state-management.md) | Redux, Zustand, Context |
| 15 | [Testing](./15-testing.md) | Vitest + React Testing Library |
| 16 | [NgModules (legacy) & architecture](./16-ngmodules-and-architecture.md) | project structure |
| 17 | [CLI, build, SSR & deployment](./17-cli-build-ssr.md) | Vite / Next.js |

### Phase 3 — Consolidate (3–5 days)
- **[18 — AngularJS → Angular → React cheat sheet](./18-cheatsheet-angularjs-angular-react.md)**: one big lookup table.
- **[19 — Interview questions & answers](./19-interview-questions.md)**: 95 questions, grouped by topic.
- **[20 — Mini project: Task Manager](./20-mini-project-task-manager.md)**: one app that uses everything (v22 style).

---

## Quick setup

```bash
# Node LTS required (check angular.dev for the supported range)
npm install -g @angular/cli@latest
ng version

ng new my-app          # pick CSS/SCSS, SSR yes/no
cd my-app
ng serve -o            # http://localhost:4200
ng generate component features/user-card   # or: ng g c features/user-card
```

## Official resources
- Docs & tutorials: <https://angular.dev>
- Interactive playground: <https://angular.dev/playground>
- Versions & support schedule: <https://angular.dev/reference/releases>
- Release notes: <https://github.com/angular/angular/releases>
- Update guide between versions: <https://angular.dev/update-guide>

> **Version note:** this guide reflects Angular v22.2 (Oct 2026). Features marked *developer
> preview* or *experimental* (e.g. `@boundary`, router resources) can still change. Check
> angular.dev for the version you're using.
