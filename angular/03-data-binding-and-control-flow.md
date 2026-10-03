# 03 — Data Binding & Control Flow

## 1. The four binding forms

| Syntax | Direction | Name | AngularJS 1.x | React |
|---|---|---|---|---|
| `{{ expr }}` | class → view | Interpolation | `{{ expr }}` | `{expr}` |
| `[prop]="expr"` | class → view | Property binding | `ng-src`, `ng-disabled`... | `prop={expr}` |
| `(event)="handler($event)"` | view → class | Event binding | `ng-click="fn()"` | `onClick={fn}` |
| `[(prop)]="value"` | both | Two-way binding | `ng-model` | `value` + `onChange` |

Mnemonic: **`[]` = data in, `()` = events out, `[()]` = "banana in a box" = both.**

```html
<!-- Interpolation -->
<h1>Hello {{ user().name }}</h1>
<p>Total: {{ price() * qty() }}</p>

<!-- Property binding: binds to DOM *properties*, not HTML attributes -->
<img [src]="avatarUrl()" [alt]="user().name" />
<button [disabled]="saving()">Save</button>
<app-user-card [user]="selectedUser()" />

<!-- Event binding -->
<button (click)="save()">Save</button>
<input (input)="query.set($any($event.target).value)" />
<input (keyup.enter)="search()" />          <!-- key event filtering -->
<app-user-card (select)="onSelect($event)" /> <!-- $event = emitted value -->

<!-- Two-way binding (needs FormsModule for ngModel) -->
<input [(ngModel)]="name" />
<app-toggle [(checked)]="isOn" />            <!-- custom two-way via model(), see ch.04 -->
```

### Attribute, class and style bindings
```html
<!-- attributes that have no DOM property (aria-*, colspan, data-*) -->
<td [attr.colspan]="span()"></td>
<button [attr.aria-label]="label()"></button>

<!-- classes -->
<div [class.active]="isActive()"></div>                         <!-- one class -->
<div [class]="{ active: isActive(), disabled: isDisabled() }"></div>  <!-- many -->
<div [class]="'card ' + theme()"></div>

<!-- styles -->
<div [style.width.px]="width()"></div>
<div [style.background-color]="color()"></div>
<div [style]="{ width: width() + 'px', opacity: 0.5 }"></div>
```
Legacy: `[ngClass]="{...}"` and `[ngStyle]="{...}"` still work and are common in older code.

React equivalent: `className={clsx({ active: isActive })}` and `style={{ width }}`.

### Template expression rules
- Can read class members, template variables, call methods, use pipes, `?.`, `??`, ternaries,
  template literals, and (since **v22**) **arrow functions** and **spread** (`...`).
- **Can't** use `new`, `++`, assignments (except in event bindings), globals (`window`, `Math`),
  or multi-statement logic. Keep logic in the class (use `computed`).
- Avoid calling expensive *plain methods* in bindings; they run on every change detection pass.
  Signals and `computed` are cheap because they're memoized.

## 2. Built-in control flow (v17+) — the modern way

### `@if` ≈ React `{cond && ...}` / ternary; AngularJS `ng-if`
```html
@if (user(); as u) {
  <p>Welcome, {{ u.name }}</p>
} @else if (loading()) {
  <app-spinner />
} @else {
  <a routerLink="/login">Log in</a>
}
```
`as u` stores the (truthy) value in a local variable, which is handy for signals and async values.

### `@for` ≈ React `.map()`; AngularJS `ng-repeat`
```html
<ul>
  @for (todo of todos(); track todo.id; let i = $index, last = $last) {
    <li [class.done]="todo.done">{{ i + 1 }}. {{ todo.title }} @if (last) { (last) }</li>
  } @empty {
    <li>No todos yet 🎉</li>
  }
</ul>
```
- `track` is **required** (the React `key`). Use a stable id; use `track $index` only for static lists.
- Implicit variables: `$index`, `$first`, `$last`, `$even`, `$odd`, `$count`.
- `@empty` renders when the list is empty (no more `@if (list.length === 0)`).

React:
```tsx
{todos.length === 0 ? <li>No todos</li> : todos.map((t, i) => <li key={t.id}>{i + 1}. {t.title}</li>)}
```

### `@switch` ≈ `switch` / object-map in JSX; AngularJS `ng-switch`
```html
@switch (status()) {
  @case ('loading') { <app-spinner /> }
  @case ('error')   { <app-error [message]="error()" /> }
  @case ('success') { <app-results [data]="data()" /> }
  @default          { <p>Idle</p> }
}
```
No fall-through, no `break` needed.

**v22:** several cases can share one block, and `@default never;` gives a **compile-time
exhaustiveness check** for union types (like TypeScript's `never` trick in a `switch`):
```html
@switch (status()) {           <!-- status: Signal<'pending' | 'processing' | 'shipped'> -->
  @case ('pending')
  @case ('processing') { <p>In progress</p> }
  @case ('shipped') { <p>On its way</p> }
  @default never;              <!-- add 'cancelled' to the union and this fails to compile -->
}
```

### `@let` (v18.1+) — local template variables
```html
@let fullName = user().first + ' ' + user().last;
@let total = cart().items.length;
<h2>{{ fullName }} ({{ total }} items)</h2>
```
React equivalent: a `const` at the top of the render function.

## 3. `@defer` — lazy-load part of a template

Something React can only do with `React.lazy` + `Suspense` + an IntersectionObserver hook. In
Angular it's built in. Components inside `@defer` are split into a separate JS chunk.

```html
@defer (on viewport; prefetch on idle) {
  <app-comments [postId]="postId()" />     <!-- loaded when scrolled into view -->
} @placeholder (minimum 300ms) {
  <div class="skeleton">Comments</div>     <!-- shown before loading starts -->
} @loading (after 100ms; minimum 500ms) {
  <app-spinner />                          <!-- shown while chunk downloads -->
} @error {
  <p>Couldn't load comments.</p>
}
```

Triggers: `on idle` (default), `on viewport`, `on interaction`, `on hover`, `on immediate`,
`on timer(2s)`, `when someCondition()`. Prefetch triggers use the same set.

> Deferred components must be standalone and **not referenced elsewhere** in the same file, or
> they'll be eagerly bundled.

## 3b. `@boundary` — error boundaries (v22.2, developer preview)

The equivalent of React Error Boundaries, as template syntax:
```html
@boundary {
  <app-recommendations />        <!-- if this (or its children) throws while rendering... -->
} @error (let err) {
  <p>Recommendations are unavailable right now.</p>   <!-- ...this renders instead -->
}
<app-checkout />                 <!-- the rest of the page keeps working -->
```
Before this, Angular only had a global `ErrorHandler`, so one failing component could break a whole
view.

## 3c. Other v22 template conveniences

```html
<!-- arrow functions -->
<button (click)="qty.update(q => q + 1)">+</button>
<app-list [sortFn]="(a, b) => a.name.localeCompare(b.name)" />

<!-- spread in object/array literals and calls -->
<div [class]="{ ...baseClasses(), selected: isSelected() }"></div>
<app-tags [tags]="[...defaultTags, 'new']" />

<!-- comments inside an element's attribute list -->
<input
  // user's work email
  type="email"
  [formField]="f.email" />
```
Keep inline functions tiny. Real logic still belongs in the class.

## 4. Legacy structural directives (you'll still read these a lot)

```html
<!-- *ngIf with else -->
<div *ngIf="user; else loadingTpl">{{ user.name }}</div>
<ng-template #loadingTpl><app-spinner /></ng-template>

<!-- *ngIf with async pipe + alias -->
<div *ngIf="user$ | async as user">{{ user.name }}</div>

<!-- *ngFor with trackBy -->
<li *ngFor="let todo of todos; let i = index; trackBy: trackById">{{ i }} {{ todo.title }}</li>
<!-- in class: trackById(index: number, t: Todo) { return t.id; } -->

<!-- ngSwitch -->
<div [ngSwitch]="status">
  <p *ngSwitchCase="'loading'">Loading…</p>
  <p *ngSwitchDefault>Idle</p>
</div>
```
These require importing `NgIf`, `NgFor`, `NgSwitch` (or `CommonModule`). The `*` is sugar for
wrapping the element in `<ng-template>`. Migrate with:
```bash
ng generate @angular/core:control-flow
```

## 5. Show/hide vs add/remove

| Want | AngularJS | Angular | React |
|---|---|---|---|
| Remove from DOM | `ng-if` | `@if` | `{cond && <X/>}` |
| Keep in DOM, hide | `ng-show` / `ng-hide` | `[hidden]="!cond"` or `[class.hidden]` | `style={{display: cond ? '' : 'none'}}` |

## 6. Event binding details

```html
<form (submit)="onSubmit($event)">           <!-- $event is the DOM Event -->
<input (keydown.shift.enter)="newLine()" />   <!-- key combos -->
<div (window:resize)="onResize()"></div>      <!-- legacy global events; prefer host: { '(window:resize)': ... } -->
```
```ts
onSubmit(e: SubmitEvent) { e.preventDefault(); }
```
Unlike React there's no synthetic event system: `$event` is the native DOM event, or the emitted
value for component outputs.

## 7. Full example — a filterable list

```ts
import { Component, computed, signal } from '@angular/core';
import { CurrencyPipe } from '@angular/common';

interface Product { id: number; name: string; price: number; inStock: boolean }

@Component({
  selector: 'app-product-list',
  template: `
    <input placeholder="Search" [value]="query()" (input)="query.set($any($event.target).value)" />
    <label><input type="checkbox" [checked]="onlyInStock()" (change)="onlyInStock.set(!onlyInStock())" /> In stock only</label>

    @let count = filtered().length;
    <p>{{ count }} result{{ count === 1 ? '' : 's' }}</p>

    <ul>
      @for (p of filtered(); track p.id) {
        <li [class.out]="!p.inStock">{{ p.name }} — {{ p.price | currency }}</li>
      } @empty {
        <li>No matches for "{{ query() }}"</li>
      }
    </ul>
  `,
  imports: [CurrencyPipe],
  styles: `.out { opacity: .5; text-decoration: line-through; }`,
})
export class ProductList {
  products = signal<Product[]>([
    { id: 1, name: 'Keyboard', price: 49, inStock: true },
    { id: 2, name: 'Mouse', price: 19, inStock: false },
    { id: 3, name: 'Monitor', price: 199, inStock: true },
  ]);
  query = signal('');
  onlyInStock = signal(false);

  filtered = computed(() => {
    const q = this.query().toLowerCase();
    return this.products().filter(p =>
      p.name.toLowerCase().includes(q) && (!this.onlyInStock() || p.inStock)
    );
  });
}
```

React equivalent: `useState` for `query`/`onlyInStock`, `useMemo` for `filtered` with
`[products, query, onlyInStock]` deps. In Angular, `computed` tracks dependencies automatically.

## 8. Interview questions

1. **Property binding vs interpolation?** Interpolation turns values into strings in text/attributes;
   property binding sets a DOM/component property to any value (objects, booleans).
2. **Property vs attribute binding?** `[value]` sets the DOM property. `[attr.colspan]` sets the
   HTML attribute, which you need when there's no matching property (aria, colspan, svg).
3. **Why is `track` mandatory in `@for`?** So Angular can reuse DOM nodes when the list changes
   instead of re-creating them. It's the same purpose as React's `key`.
4. **What does `@defer` do?** It lazy-loads a template block and its dependencies into a separate
   chunk, based on triggers like viewport or idle, with placeholder/loading/error states.
5. **How does two-way binding work?** `[(x)]="v"` is sugar for `[x]="v" (xChange)="v = $event"`.
6. **New control flow vs `*ngIf`?** Built-in syntax: no import needed, better type narrowing,
   `@empty`, faster, and required `track`.
7. **Does Angular have error boundaries?** Yes, `@boundary { } @error { }` since v22.2 (developer
   preview). Before that, only the global `ErrorHandler`.
8. **What's new in templates in v22?** Arrow functions, spread syntax, multi-case `@case`,
   exhaustive `@default never;`, and comments inside element tags.
