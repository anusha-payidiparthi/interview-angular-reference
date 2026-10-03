# 06 — Directives & Pipes

## 1. Three kinds of directives

| Kind | What it does | Examples |
|---|---|---|
| **Component** | Directive *with a template* | Every `@Component` |
| **Attribute directive** | Changes appearance/behavior of an existing element | `ngClass`, `ngModel`, `routerLink`, your `appHighlight` |
| **Structural directive** | Adds/removes DOM by rendering templates | `*ngIf`, `*ngFor`, `*ngTemplateOutlet`, your `*appPermission` |

AngularJS: everything was `.directive()` with `restrict: 'EA'`, `link`, `compile`. Angular splits it:
**components** for UI, **directives** for reusable behavior on elements.

React: no direct equivalent. You'd use a **custom hook + ref** (`useClickOutside(ref)`), a
wrapper component, or a HOC. Angular directives attach to *any* element without wrapping it.

## 2. Attribute directive

```ts
// highlight.directive.ts
import { Directive, ElementRef, inject, input } from '@angular/core';

@Directive({
  selector: '[appHighlight]',          // attribute selector
  host: {
    '(mouseenter)': 'setColor(color())',
    '(mouseleave)': 'setColor("")',
    '[style.cursor]': '"pointer"',
  },
})
export class Highlight {
  color = input('yellow', { alias: 'appHighlight' });   // <p appHighlight="pink">
  private el = inject(ElementRef<HTMLElement>);

  protected setColor(c: string) {
    this.el.nativeElement.style.backgroundColor = c;
  }
}
```
```html
<p appHighlight>Default yellow</p>
<p appHighlight="lightblue">Blue on hover</p>
```
Add `Highlight` to the host component's `imports`.

Legacy equivalent of `host`:
```ts
@HostListener('mouseenter') onEnter() { ... }
@HostBinding('style.cursor') cursor = 'pointer';
```

### A practical one: click outside
```ts
@Directive({
  selector: '[appClickOutside]',
  host: { '(document:click)': 'onDocClick($event)' },
})
export class ClickOutside {
  appClickOutside = output<void>();
  private el = inject(ElementRef);

  protected onDocClick(e: MouseEvent) {
    if (!this.el.nativeElement.contains(e.target)) this.appClickOutside.emit();
  }
}
```
```html
<div class="dropdown" (appClickOutside)="open.set(false)">...</div>
```
React: `useClickOutside(ref, () => setOpen(false))`.

## 3. Structural directive

A structural directive gets a `TemplateRef` (the "what") and a `ViewContainerRef` (the "where").

```ts
// has-role.directive.ts — show element only if the user has a role
import { Directive, TemplateRef, ViewContainerRef, effect, inject, input } from '@angular/core';
import { AuthService } from './auth.service';

@Directive({ selector: '[appHasRole]' })
export class HasRole {
  appHasRole = input.required<string>();
  private tpl = inject(TemplateRef<unknown>);
  private vcr = inject(ViewContainerRef);
  private auth = inject(AuthService);
  private rendered = false;

  constructor() {
    effect(() => {
      const allowed = this.auth.roles().includes(this.appHasRole());
      if (allowed && !this.rendered) {
        this.vcr.createEmbeddedView(this.tpl);
        this.rendered = true;
      } else if (!allowed && this.rendered) {
        this.vcr.clear();
        this.rendered = false;
      }
    });
  }
}
```
```html
<button *appHasRole="'admin'">Delete user</button>
<!-- the * desugars to: -->
<ng-template [appHasRole]="'admin'"><button>Delete user</button></ng-template>
```

React equivalent: `<HasRole role="admin"><button/></HasRole>` or `{hasRole('admin') && <button/>}`.

## 4. Directive composition API (`hostDirectives`, v15+)

Reuse behavior across components without inheritance (closest React analogy: composing hooks).

```ts
@Directive({ selector: '[appFocusRing]', host: { '[class.focus-ring]': 'true' } })
export class FocusRing {}

@Directive({ selector: '[appTooltip]' })
export class Tooltip { text = input(''); }

@Component({
  selector: 'app-icon-button',
  template: `<ng-content />`,
  hostDirectives: [
    FocusRing,
    { directive: Tooltip, inputs: ['text: tooltip'] },  // expose Tooltip's input as [tooltip]
  ],
})
export class IconButton {}
```
```html
<app-icon-button tooltip="Save">💾</app-icon-button>
```

## 5. Pipes

Pipes transform values **in templates**. They're the successor to AngularJS **filters**, with the
same `|` syntax.

### Built-in pipes (import from `@angular/common`)
```html
{{ birthday | date: 'longDate' }}           <!-- June 15, 1990 -->
{{ birthday | date: 'yyyy-MM-dd HH:mm' }}
{{ price | currency: 'EUR' }}               <!-- €49.00 -->
{{ ratio | percent: '1.0-1' }}              <!-- 45.3% -->
{{ pi | number: '1.2-2' }}                  <!-- 3.14 -->
{{ name | uppercase }} {{ name | lowercase }} {{ title | titlecase }}
{{ obj | json }}                            <!-- debugging -->
{{ items | slice: 0:5 }}
@for (entry of map | keyvalue; track entry.key) { {{ entry.key }}={{ entry.value }} }
{{ user$ | async }}                         <!-- subscribes & unsubscribes for you -->
```
Chain them: `{{ birthday | date: 'fullDate' | uppercase }}`.

React equivalent: call formatter functions in JSX: `{formatDate(birthday)}`, `Intl.NumberFormat`.

### Custom pipe
```ts
// truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'truncate' })               // standalone by default (v19+)
export class TruncatePipe implements PipeTransform {
  transform(value: string | null | undefined, limit = 20, suffix = '…'): string {
    if (!value) return '';
    return value.length > limit ? value.slice(0, limit) + suffix : value;
  }
}
```
```html
{{ post.body | truncate: 100 }}
{{ post.body | truncate: 50: ' [more]' }}
```

### Pure vs impure pipes
- **Pure (default):** re-runs only when the **input reference** changes. Memoized and fast. Like
  `useMemo` keyed on the arguments.
- **Impure (`pure: false`):** re-runs on **every change detection pass**. Needed if you depend on
  mutated arrays/objects or external state. `async` is impure. Use sparingly.

```ts
@Pipe({ name: 'filterBy', pure: false })   // re-evaluates even if the array is mutated in place
export class FilterByPipe implements PipeTransform { ... }
```
**Prefer `computed()` over filtering pipes.** AngularJS had `filter` and `orderBy` filters;
Angular intentionally doesn't ship them for performance reasons.

### Pipes with DI
Pipes can `inject()` services (e.g. a `TranslatePipe` injecting a translation service).

## 6. When to use what (React dev decision guide)

| You'd write in React | Write in Angular |
|---|---|
| A component that renders UI | Component |
| A custom hook with state/logic, no DOM | **Service** (or plain function with `inject()`) |
| A custom hook that attaches to a DOM element via ref | **Attribute directive** |
| Conditional/repeat wrapper component | **Structural directive** or control flow |
| A formatting helper used in JSX | **Pipe** |
| A HOC adding behavior | `hostDirectives` |

## 7. Gotchas

1. Avoid touching `nativeElement` directly where you can. Prefer `host` bindings, which are
   SSR-safe and declarative. Use `Renderer2` if you must manipulate the DOM imperatively in older code.
2. Only one structural directive per element (`*ngIf` + `*ngFor` on the same tag is an error).
   Wrap with `<ng-container>`. With `@if`/`@for` this isn't an issue.
3. A pipe argument that's an object created inline (`| fn: {a: 1}`) creates a new reference on every check.

## 8. Interview questions

1. **Component vs directive?** A component is a directive with a template. Directives add
   behavior to existing elements.
2. **Attribute vs structural directive?** Attribute directives modify an element; structural
   directives add or remove DOM using `TemplateRef` + `ViewContainerRef` (the `*` syntax).
3. **What is `HostListener`/`HostBinding`?** Decorators to listen to events on, and bind
   properties of, the host element. The modern equivalent is the `host` metadata object.
4. **Pure vs impure pipe?** A pure pipe recomputes only when its input references change; an
   impure pipe recomputes every change detection cycle.
5. **Why no `filter`/`orderBy` pipes like AngularJS?** They'd need to be impure and would run
   constantly, which hurts performance. Filter in the component (`computed`).
6. **What does the `async` pipe do?** Subscribes to an Observable/Promise, returns the latest
   value, marks the component for check on emission, and unsubscribes on destroy.
