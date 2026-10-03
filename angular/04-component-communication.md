# 04 — Component Communication

## 1. Overview

| Need | AngularJS 1.x | React | Angular (modern) | Angular (legacy) |
|---|---|---|---|---|
| Parent → child data | `bindings: { x: '<' }` | props | `input()` | `@Input()` |
| Child → parent event | `bindings: { onX: '&' }` | callback prop | `output()` | `@Output() EventEmitter` |
| Two-way | `bindings: { x: '=' }` | value + onChange | `model()` | `@Input() x` + `@Output() xChange` |
| Parent calls child method | `require` / scope hacks | `useRef` + `forwardRef` + `useImperativeHandle` | `viewChild()` | `@ViewChild()` |
| Access projected children | transclusion | `React.Children` | `contentChild(ren)()` | `@ContentChild(ren)` |
| Siblings / distant components | `$rootScope.$broadcast`, shared service | Context / store | Shared **service** (signals/RxJS) | same |

## 2. Inputs — `input()`

```ts
import { Component, input, booleanAttribute, numberAttribute, computed } from '@angular/core';

@Component({
  selector: 'app-avatar',
  template: `<img [src]="url()" [width]="size()" [class.round]="rounded()" [alt]="alt()" />`,
})
export class Avatar {
  url = input.required<string>();                    // required: compile error if missing
  size = input(48, { transform: numberAttribute });  // default 48; "64" string → 64
  rounded = input(false, { transform: booleanAttribute }); // <app-avatar rounded /> → true
  alt = input('', { alias: 'label' });               // used as [label]="..."

  // derived from inputs: just use computed (React: compute in render / useMemo)
  big = computed(() => this.size() > 100);
}
```
```html
<app-avatar [url]="user().photo" size="64" rounded label="Profile photo" />
```

- An input is a **read-only signal**: read with `this.size()`, and you can't `.set()` it.
- To react to changes, use `computed()` or `effect()` — no `ngOnChanges` needed.

### Legacy `@Input()`
```ts
export class Avatar implements OnChanges {
  @Input({ required: true }) url!: string;
  @Input() size = 48;
  @Input({ transform: booleanAttribute }) rounded = false;

  // setter style, to react to each change
  @Input() set theme(value: string) { this.applyTheme(value); }

  ngOnChanges(changes: SimpleChanges) {
    if (changes['size']) console.log(changes['size'].previousValue, '→', changes['size'].currentValue);
  }
}
```

## 3. Outputs — `output()`

```ts
import { Component, input, output } from '@angular/core';

@Component({
  selector: 'app-todo-item',
  template: `
    <input type="checkbox" [checked]="todo().done" (change)="toggle.emit(todo().id)" />
    {{ todo().title }}
    <button (click)="remove.emit(todo().id)">✕</button>
  `,
})
export class TodoItem {
  todo = input.required<{ id: number; title: string; done: boolean }>();
  toggle = output<number>();
  remove = output<number>();
}
```
```html
<!-- parent -->
<app-todo-item [todo]="t" (toggle)="onToggle($event)" (remove)="onRemove($event)" />
```

React:
```tsx
<TodoItem todo={t} onToggle={onToggle} onRemove={onRemove} />
```

> In React you pass a *function* down. In Angular the child *emits an event* and the parent
> *listens*. Outputs don't bubble through the DOM; only the direct parent can listen.

### Legacy `@Output()`
```ts
@Output() remove = new EventEmitter<number>();
// emit the same way: this.remove.emit(id)
```
`EventEmitter` extends RxJS `Subject`. Use it only for outputs, never as a general event bus.

### Outputs from observables
```ts
import { outputFromObservable } from '@angular/core/rxjs-interop';
search = outputFromObservable(this.searchTerms$.pipe(debounceTime(300)));
```

## 4. Two-way binding — `model()`

```ts
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-toggle',
  template: `<button (click)="checked.set(!checked())">{{ checked() ? 'ON' : 'OFF' }}</button>`,
})
export class Toggle {
  checked = model(false);   // writable signal + automatically creates a "checkedChange" output
}
```
```ts
// parent
@Component({
  imports: [Toggle],
  template: `
    <app-toggle [(checked)]="darkMode" />   <!-- pass the signal itself, no () -->
    <p>Dark mode is {{ darkMode() ? 'on' : 'off' }}</p>
  `,
})
export class Settings {
  darkMode = signal(false);
}
```

React equivalent: `<Toggle checked={dark} onCheckedChange={setDark} />`.
AngularJS equivalent: `bindings: { checked: '=' }`.

Legacy pattern (what `model()` replaced):
```ts
@Input() checked = false;
@Output() checkedChange = new EventEmitter<boolean>();   // name MUST be <input>Change
```

## 5. Parent → child imperative access — `viewChild()`

```ts
import { Component, ElementRef, viewChild, afterNextRender } from '@angular/core';

@Component({
  selector: 'app-search-page',
  imports: [VideoPlayer],
  template: `
    <input #searchBox />
    <app-video-player />
    <button (click)="player().play()">Play</button>
  `,
})
export class SearchPage {
  searchBox = viewChild.required<ElementRef<HTMLInputElement>>('searchBox'); // by template ref
  player = viewChild.required(VideoPlayer);                                  // by component type

  constructor() {
    afterNextRender(() => this.searchBox().nativeElement.focus());   // DOM is ready
  }
}
```
- `viewChild()` returns a signal (`Signal<T | undefined>`); `.required` makes it `Signal<T>`.
- `viewChildren(Type)` returns `Signal<readonly T[]>`.

React: `const ref = useRef<HTMLInputElement>(null); useEffect(() => ref.current?.focus(), []);`

Legacy:
```ts
@ViewChild('searchBox') searchBox!: ElementRef<HTMLInputElement>;     // available in ngAfterViewInit
@ViewChild('searchBox', { static: true }) ...                          // available in ngOnInit
@ViewChildren(TodoItem) items!: QueryList<TodoItem>;
```

## 6. Accessing projected content — `contentChild()` / `contentChildren()`

```ts
@Component({
  selector: 'app-tabs',
  template: `
    <nav>
      @for (tab of tabs(); track tab.title()) {
        <button [class.active]="tab === active()" (click)="active.set(tab)">{{ tab.title() }}</button>
      }
    </nav>
    <ng-content />
  `,
})
export class Tabs {
  tabs = contentChildren(Tab);                        // Tab components projected inside <app-tabs>
  active = linkedSignal(() => this.tabs()[0]);        // default to first tab, user can change it
}

@Component({
  selector: 'app-tab',
  template: `@if (isActive()) { <ng-content /> }`,
})
export class Tab {
  title = input.required<string>();
  private tabs = inject(Tabs);                        // child can inject its parent component!
  isActive = computed(() => this.tabs.active() === this);
}
```
```html
<app-tabs>
  <app-tab title="Profile">Profile content</app-tab>
  <app-tab title="Settings">Settings content</app-tab>
</app-tabs>
```
This is the **compound component** pattern (React: Context shared by `<Tabs>` and `<Tab>`). In
Angular a child can `inject()` an ancestor component directly.

## 7. Sibling / cross-tree communication — a shared service

The Angular answer to React Context or a Zustand store:

```ts
// cart.store.ts
import { Injectable, computed, signal } from '@angular/core';

export interface CartItem { id: number; name: string; price: number; qty: number }

@Injectable({ providedIn: 'root' })        // app-wide singleton
export class CartStore {
  private readonly _items = signal<CartItem[]>([]);
  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price * i.qty, 0));
  readonly count = computed(() => this._items().reduce((s, i) => s + i.qty, 0));

  add(item: Omit<CartItem, 'qty'>) {
    this._items.update(items => {
      const existing = items.find(i => i.id === item.id);
      return existing
        ? items.map(i => (i.id === item.id ? { ...i, qty: i.qty + 1 } : i))
        : [...items, { ...item, qty: 1 }];
    });
  }

  remove(id: number) {
    this._items.update(items => items.filter(i => i.id !== id));
  }
}
```
```ts
// header.ts — anywhere in the tree
@Component({ selector: 'app-header', template: `🛒 {{ cart.count() }}` })
export class Header { protected cart = inject(CartStore); }

// product.ts — anywhere else
@Component({
  selector: 'app-product',
  template: `<button (click)="cart.add(product())">Add</button>`,
})
export class Product {
  product = input.required<{ id: number; name: string; price: number }>();
  protected cart = inject(CartStore);
}
```
No provider wrapper and no prop drilling. Change the provider scope to limit sharing to a subtree
(see [chapter 07](./07-dependency-injection.md)).

## 8. Gotchas

1. **Inputs are immutable from the child.** Don't try to `.set()` an `input()`. Use `model()` for
   two-way, or copy into a `linkedSignal` for local editable state.
2. **Mutating objects passed as inputs** won't trigger `OnPush`/signal updates. Pass new references
   (immutability, same rule as React).
3. **Output naming:** don't prefix with `on` (`(onSave)`); the convention is `(save)`. For
   two-way binding, the output must be named `<input>Change`.
4. **`viewChild` is undefined until the view renders.** Read it in `afterNextRender`, an `effect`,
   or event handlers, not in the constructor.

## 9. Interview questions

1. **How do components communicate?** Inputs/outputs (parent↔child), `model()` for two-way, view
   and content queries for direct access, and shared services for unrelated components.
2. **`input()` vs `@Input()`?** `input()` returns a read-only signal, which works with `computed`/`effect`
   and removes the need for `ngOnChanges` and setters. `@Input()` is a plain property.
3. **What is `model()`?** A writable signal input that also emits `<name>Change`, enabling `[( )]`.
4. **`ViewChild` vs `ContentChild`?** ViewChild queries elements in the component's own template;
   ContentChild queries elements projected into it via `<ng-content>`.
5. **What's `{ static: true }` on `@ViewChild`?** It resolves the query before the first change
   detection (usable in `ngOnInit`), which only works if the element isn't inside `@if`/`@for`.
6. **Why not use `EventEmitter` in services?** It's intended for `@Output`. Use `Subject` or signals.
