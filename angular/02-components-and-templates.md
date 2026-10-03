# 02 — Components & Templates

## 1. Concept

A component is a **TypeScript class** + an **HTML template** + optional **styles**, tied together
with the `@Component` decorator. Angular creates one instance per element on the page and keeps it
alive until the element is removed.

| | AngularJS 1.x | React | Angular |
|---|---|---|---|
| Unit of UI | `.component()` / `.directive()` + controller | Function returning JSX | Class + template |
| State | `this` on controller / `$scope` | `useState` | class fields / `signal()` |
| Template | HTML string or `templateUrl` | JSX | HTML string or `templateUrl` |
| Styles | global CSS | CSS modules / CSS-in-JS | Scoped per component by default |
| Re-render | digest cycle | function re-runs | template bindings re-checked; class not re-created |

## 2. Anatomy of a component

```ts
// user-card.ts   (pre-v20 style guide name: user-card.component.ts)
import { Component, ChangeDetectionStrategy, input, output, computed } from '@angular/core';
import { DatePipe, UpperCasePipe } from '@angular/common';

export interface User {
  id: number;
  name: string;
  email: string;
  joined: Date;
}

@Component({
  selector: 'app-user-card',           // how you use it: <app-user-card />
  imports: [DatePipe, UpperCasePipe],  // what the TEMPLATE uses (components, directives, pipes)
  templateUrl: './user-card.html',     // or inline: template: `...`
  styleUrl: './user-card.css',         // or styles: `...`  (styleUrls: [] in older versions)
  changeDetection: ChangeDetectionStrategy.OnPush,  // the default in v22 (can omit); needed explicitly in ≤ v21
  host: { class: 'card', '[class.selected]': 'selected()' }, // bindings on the host element
})
export class UserCard {
  user = input.required<User>();
  selected = input(false);
  select = output<number>();

  protected initials = computed(() =>
    this.user().name.split(' ').map(p => p[0]).join('')
  );

  protected onClick() {
    this.select.emit(this.user().id);
  }
}
```

```html
<!-- user-card.html -->
<div class="avatar">{{ initials() }}</div>
<h3>{{ user().name | uppercase }}</h3>
<p>{{ user().email }}</p>
<small>Joined {{ user().joined | date: 'mediumDate' }}</small>
<button (click)="onClick()">Select</button>
```

```css
/* user-card.css — scoped to this component only */
:host { display: block; border: 1px solid #ddd; padding: 1rem; }
:host(.selected) { border-color: royalblue; }
h3 { margin: 0; }   /* won't leak to other h3s */
```

Usage in a parent:
```ts
@Component({
  selector: 'app-user-list',
  imports: [UserCard],              // must import child components you use
  template: `
    @for (u of users(); track u.id) {
      <app-user-card [user]="u" [selected]="u.id === selectedId()" (select)="selectedId.set($event)" />
    }
  `,
})
export class UserList {
  users = signal<User[]>([]);
  selectedId = signal<number | null>(null);
}
```

React equivalent:
```tsx
function UserCard({ user, selected = false, onSelect }: Props) {
  const initials = useMemo(() => user.name.split(' ').map(p => p[0]).join(''), [user.name]);
  return (
    <div className={`card ${selected ? 'selected' : ''}`}>
      <div className="avatar">{initials}</div>
      <h3>{user.name.toUpperCase()}</h3>
      <button onClick={() => onSelect(user.id)}>Select</button>
    </div>
  );
}
```

### `@Component` metadata you should know

| Property | Purpose |
|---|---|
| `selector` | Element name (`app-x`), attribute (`[appX]`) or class (`.x`). Prefix with `app-` or your team prefix |
| `imports` | Components/directives/pipes the template uses (standalone components) |
| `template` / `templateUrl` | Inline or external HTML |
| `styles` / `styleUrl(s)` | Inline or external CSS |
| `changeDetection` | `OnPush` (default in v22) or `Eager` (called `Default` before v22) |
| `encapsulation` | `Emulated` (default), `ShadowDom`, `None` |
| `providers` / `viewProviders` | Component-scoped services (chapter 07) |
| `host` | Attributes, classes, listeners on the host element |
| `hostDirectives` | Compose directives onto this component (chapter 06) |
| `standalone` | `true` by default since v19. `false` = declared in an NgModule (legacy) |

## 3. Standalone vs NgModule components

```ts
// Modern (v19+ default): standalone — the component lists its own dependencies
@Component({ selector: 'app-a', imports: [CommonThing, OtherComponent], template: '...' })
export class A {}

// Legacy: declared in an NgModule; dependencies come from the module's imports
@Component({ selector: 'app-a', template: '...', standalone: false })
export class A {}

@NgModule({ declarations: [A], imports: [CommonModule], exports: [A] })
export class FeatureModule {}
```
See [chapter 16](./16-ngmodules-and-architecture.md) for NgModules.

## 4. Bootstrapping the app

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { App } from './app/app';

bootstrapApplication(App, appConfig).catch(err => console.error(err));
```

```ts
// app.config.ts
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    // zoneless is the default since v21; v20 apps add provideZonelessChangeDetection() here
    provideRouter(routes),
    provideHttpClient(withFetch()),
  ],
};
```

```html
<!-- index.html -->
<body><app-root></app-root></body>
```

React equivalent: `createRoot(document.getElementById('root')).render(<Providers><App/></Providers>)`.

## 5. Content projection (`ng-content`) ≈ React `children`

### Single slot
```ts
@Component({
  selector: 'app-panel',
  template: `
    <section class="panel">
      <ng-content />            <!-- whatever the parent puts between the tags -->
    </section>
  `,
})
export class Panel {}
```
```html
<app-panel><p>Hello inside the panel</p></app-panel>
```
React: `function Panel({ children }) { return <section>{children}</section> }`

### Multi-slot (named slots) ≈ React "render props as named props"
```ts
@Component({
  selector: 'app-card',
  template: `
    <header><ng-content select="[card-title]" /></header>
    <div class="body"><ng-content /></div>              <!-- default slot -->
    <footer><ng-content select="app-card-actions" /></footer>
  `,
})
export class Card {}
```
```html
<app-card>
  <h2 card-title>Profile</h2>
  <p>Body text</p>
  <app-card-actions><button>Save</button></app-card-actions>
</app-card>
```
React: `<Card title={<h2>Profile</h2>} actions={<button>Save</button>}>Body</Card>`

AngularJS: `transclude: true` + `ng-transclude` — same idea, new name.

### Fallback content (v18+)
```html
<ng-content select="[icon]">⭐</ng-content>   <!-- shown if nothing is projected -->
```

> **Gotcha:** projected content is always instantiated by the parent, even if the child wraps
> `<ng-content>` in an `@if`. To render content *conditionally/lazily* or *multiple times*, the
> child should accept an `ng-template` instead (see below).

## 6. `ng-template`, `ng-container`, template reference variables

```html
<!-- ng-container: groups elements without adding a DOM node — like React <></> -->
<ng-container>
  <td>{{ a }}</td><td>{{ b }}</td>
</ng-container>

<!-- ng-template: a chunk of template that is NOT rendered until something renders it -->
<ng-template #loading><spinner /></ng-template>

<!-- render it with ngTemplateOutlet (import NgTemplateOutlet) -->
<ng-container *ngTemplateOutlet="loading" />

<!-- pass context, like a render prop: renderItem={(item) => ...} -->
<ng-template #row let-item let-i="index">
  <li>{{ i }}: {{ item.name }}</li>
</ng-template>
<ng-container *ngTemplateOutlet="row; context: { $implicit: user, index: 0 }" />
```

Render-prop style component (child decides where/when to render a parent-supplied template):
```ts
@Component({
  selector: 'app-list',
  imports: [NgTemplateOutlet],
  template: `
    <ul>
      @for (item of items(); track $index) {
        <ng-container *ngTemplateOutlet="itemTemplate(); context: { $implicit: item }" />
      }
    </ul>
  `,
})
export class List<T> {
  items = input.required<T[]>();
  itemTemplate = input.required<TemplateRef<{ $implicit: T }>>();
}
```
```html
<ng-template #userTpl let-user><li>{{ user.name }}</li></ng-template>
<app-list [items]="users()" [itemTemplate]="userTpl" />
```

**Template reference variables** (`#name`) give the template a handle to an element, component or
directive:
```html
<input #search (keyup.enter)="find(search.value)" />
<app-video-player #player />
<button (click)="player.play()">Play</button>
```

## 7. Styling & view encapsulation

- **Emulated** (default): Angular adds attributes like `_ngcontent-abc` to scope styles. Styles
  in a component don't leak out, and global styles still leak in.
- **ShadowDom**: real Shadow DOM.
- **None**: styles become global.

Special selectors:
```css
:host { display: block; }                 /* the component's own element */
:host(.active) { ... }                    /* host has class active */
:host-context(.dark-theme) h1 { ... }     /* some ancestor has .dark-theme */
::ng-deep .mat-button { ... }             /* pierce into children — deprecated, avoid */
```
Global styles live in `src/styles.css`.

## 8. Generating components with the CLI

```bash
ng g c features/users/user-card            # creates .ts, .html, .css, .spec.ts
ng g c shared/badge --inline-template --inline-style --skip-tests
```

> **Style guide note (v20+):** the CLI now generates `user-card.ts` with class `UserCard`
> (no `.component` suffix / `Component` suffix). Older projects use `user-card.component.ts` with
> `UserCardComponent`. Both are fine — follow the project's convention.

## 9. Gotchas for React devs

1. **You must add child components to `imports`**, or Angular reports an unknown element error. In
   React, importing the function is enough. In Angular, the TS import *and* the `imports` array
   are both needed.
2. **The class is not re-run.** Don't compute derived values in the constructor and expect them to
   update. Use `computed()` or getters.
3. **Templates can only access class members** (and template variables), not module-level
   variables or globals like `Math`, `console` or `window`. Expose them as class fields.
4. **Selectors must contain a dash** for custom elements (`app-user`, not `user`).
5. **Self-closing tags** (`<app-x />`) are supported since v15.1.

## 10. Interview questions

1. **What is a component in Angular?** A class decorated with `@Component` that controls a part of
   the UI through its template; it has a selector, template, styles and optional providers.
2. **Standalone vs NgModule components?** Standalone components declare their own template
   dependencies in `imports` and don't need an NgModule. They're the default since v19.
3. **What is view encapsulation?** How Angular scopes component CSS: Emulated (attribute-based,
   default), ShadowDom, or None.
4. **What is content projection?** Passing markup from a parent into a child's `<ng-content>` slot
   (like `children`). Multi-slot uses `select`.
5. **`ng-template` vs `ng-container`?** `ng-container` is a grouping element that renders its
   children without a wrapper node. `ng-template` defines a template that renders only when
   instantiated (by a structural directive, `ngTemplateOutlet`, or `ViewContainerRef`).
6. **What does `:host` do?** Styles the component's own host element.
