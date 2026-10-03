# 20 — Mini Project: Task Manager (Modern Angular v22)

Build this end to end and you'll have touched every major feature. **Type it yourself**
rather than copy-pasting; that's where the learning happens.

**Features used:** standalone components, signals (`signal`, `computed`, `linkedSignal`, `effect`),
`input`/`output`, new control flow, `@defer`, `@Service()`, `httpResource` + `HttpClient`,
functional interceptor, lazy routes, route input binding, functional guard, Signal Forms, a custom
pipe, a custom directive, tests.

> If your company is on an older version, swap: `@Service()` → `@Injectable({ providedIn: 'root' })`;
> Signal Forms → Reactive Forms (chapter 12); `httpResource` → `HttpClient` + `toSignal`.

---

## 0. Setup

```bash
ng new task-manager --style=css --ssr=false
cd task-manager
npm i -D json-server
```

Fake REST API: `db.json` at the project root
```json
{
  "tasks": [
    { "id": "1", "title": "Learn signals", "priority": "high", "dueDate": "2026-10-05", "done": false },
    { "id": "2", "title": "Read routing chapter", "priority": "medium", "dueDate": "2026-10-08", "done": false },
    { "id": "3", "title": "Set up project", "priority": "low", "dueDate": "2026-10-01", "done": true }
  ]
}
```

`proxy.conf.json` (forwards `/api/*` to json-server, which avoids CORS)
```json
{ "/api": { "target": "http://localhost:3000", "secure": false, "pathRewrite": { "^/api": "" } } }
```

`package.json` scripts
```json
"api": "json-server db.json --port 3000",
"start": "ng serve --proxy-config proxy.conf.json"
```
Run `npm run api` and `npm start` in two terminals.

Target structure:
```
src/app/
├── app.ts  app.config.ts  app.routes.ts
├── core/
│   ├── auth.store.ts
│   ├── auth.guard.ts
│   └── auth.interceptor.ts
├── shared/
│   ├── relative-due.pipe.ts
│   └── autofocus.directive.ts
├── login/login-page.ts
└── features/tasks/
    ├── task.model.ts
    ├── task.store.ts
    ├── tasks.routes.ts
    ├── task-item.ts
    ├── task-list-page.ts
    ├── task-form-page.ts
    └── task-stats.ts
```

---

## 1. Model

```ts
// features/tasks/task.model.ts
export type Priority = 'low' | 'medium' | 'high';

export interface Task {
  id: string;
  title: string;
  priority: Priority;
  dueDate: string;     // yyyy-MM-dd
  done: boolean;
}

export type TaskDraft = Omit<Task, 'id'>;

export const emptyDraft = (): TaskDraft => ({ title: '', priority: 'medium', dueDate: '', done: false });
```

## 2. Auth: store, interceptor, guard

```ts
// core/auth.store.ts
import { Service, computed, effect, signal } from '@angular/core';

@Service()
export class AuthStore {
  private readonly _user = signal<string | null>(localStorage.getItem('user'));
  readonly user = this._user.asReadonly();
  readonly isLoggedIn = computed(() => this._user() !== null);
  readonly token = computed(() => (this._user() ? `fake-token-for-${this._user()}` : null));

  constructor() {
    effect(() => {                                   // sync to localStorage (side effect → effect)
      const u = this._user();
      u ? localStorage.setItem('user', u) : localStorage.removeItem('user');
    });
  }

  login(name: string) { this._user.set(name); }
  logout() { this._user.set(null); }
}
```

```ts
// core/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthStore } from './auth.store';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthStore).token();
  return token ? next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })) : next(req);
};
```

```ts
// core/auth.guard.ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';
import { AuthStore } from './auth.store';

export const authGuard: CanActivateFn = (_route, state) =>
  inject(AuthStore).isLoggedIn()
    ? true
    : inject(Router).createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
```

## 3. App config, routes, root component

```ts
// app.config.ts
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withFetch, withInterceptors } from '@angular/common/http';
import { routes } from './app.routes';
import { authInterceptor } from './core/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withFetch(), withInterceptors([authInterceptor])),
    // zoneless is the default for new v21+ apps; OnPush is the default for components in v22
  ],
};
```

```ts
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './core/auth.guard';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'tasks' },
  { path: 'login', title: 'Login', loadComponent: () => import('./login/login-page').then(m => m.LoginPage) },
  {
    path: 'tasks',
    canActivate: [authGuard],
    loadChildren: () => import('./features/tasks/tasks.routes').then(m => m.TASK_ROUTES),
  },
  { path: '**', redirectTo: 'tasks' },
];
```

```ts
// features/tasks/tasks.routes.ts
import { Routes } from '@angular/router';

export const TASK_ROUTES: Routes = [
  { path: '', title: 'Tasks', loadComponent: () => import('./task-list-page').then(m => m.TaskListPage) },
  { path: 'new', title: 'New task', loadComponent: () => import('./task-form-page').then(m => m.TaskFormPage) },
  { path: ':id/edit', title: 'Edit task', loadComponent: () => import('./task-form-page').then(m => m.TaskFormPage) },
];
```

```ts
// app.ts
import { Component, inject } from '@angular/core';
import { Router, RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';
import { AuthStore } from './core/auth.store';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <header>
      <a routerLink="/tasks" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: true }">Tasks</a>
      <a routerLink="/tasks/new" routerLinkActive="active">New</a>
      <span class="spacer"></span>
      @if (auth.user(); as user) {
        Hi, {{ user }} <button (click)="logout()">Log out</button>
      }
    </header>
    <main><router-outlet /></main>
  `,
  styles: `
    header { display: flex; gap: 1rem; padding: 1rem; border-bottom: 1px solid #ddd; align-items: center; }
    .spacer { flex: 1; }
    .active { font-weight: bold; }
    main { padding: 1rem; max-width: 720px; margin: auto; }
  `,
})
export class App {
  protected auth = inject(AuthStore);
  private router = inject(Router);

  logout() {
    this.auth.logout();
    this.router.navigate(['/login']);
  }
}
```

## 4. Shared pipe and directive

```ts
// shared/relative-due.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

const DAY = 24 * 60 * 60 * 1000;

@Pipe({ name: 'relativeDue' })
export class RelativeDuePipe implements PipeTransform {
  transform(date: string | null | undefined, today: Date = new Date()): string {
    if (!date) return 'No due date';
    const start = new Date(today.getFullYear(), today.getMonth(), today.getDate()).getTime();
    const diff = Math.round((new Date(date + 'T00:00:00').getTime() - start) / DAY);
    if (diff === 0) return 'Due today';
    if (diff === 1) return 'Due tomorrow';
    return diff < 0 ? `Overdue by ${-diff}d` : `Due in ${diff}d`;
  }
}
```

```ts
// shared/autofocus.directive.ts
import { Directive, ElementRef, afterNextRender, inject } from '@angular/core';

@Directive({ selector: '[appAutofocus]' })
export class Autofocus {
  constructor() {
    const el = inject<ElementRef<HTMLElement>>(ElementRef);
    afterNextRender(() => el.nativeElement.focus());
  }
}
```

## 5. Task store (signals + httpResource + HttpClient)

```ts
// features/tasks/task.store.ts
import { Service, computed, inject, signal } from '@angular/core';
import { HttpClient, httpResource } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { Task, TaskDraft } from './task.model';

export type Filter = 'all' | 'active' | 'done';

@Service()
export class TaskStore {
  private http = inject(HttpClient);

  // READ: reactive resource, refetched via reload()
  private readonly tasksResource = httpResource<Task[]>(() => '/api/tasks', { defaultValue: [] });

  readonly tasks = this.tasksResource.value;
  readonly loading = this.tasksResource.isLoading;
  readonly error = this.tasksResource.error;

  readonly filter = signal<Filter>('all');
  readonly search = signal('');

  readonly visible = computed(() => {
    const f = this.filter();
    const q = this.search().toLowerCase();
    return this.tasks()
      .filter(t => f === 'all' || (f === 'done' ? t.done : !t.done))
      .filter(t => t.title.toLowerCase().includes(q))
      .sort((a, b) => Number(a.done) - Number(b.done) || a.dueDate.localeCompare(b.dueDate));
  });

  readonly stats = computed(() => {
    const all = this.tasks();
    const done = all.filter(t => t.done).length;
    return { total: all.length, done, open: all.length - done, pct: all.length ? Math.round((done / all.length) * 100) : 0 };
  });

  // WRITES: HttpClient, then refresh
  async create(draft: TaskDraft) {
    await firstValueFrom(this.http.post<Task>('/api/tasks', draft));
    this.tasksResource.reload();
  }

  async update(id: string, changes: Partial<Task>) {
    await firstValueFrom(this.http.patch<Task>(`/api/tasks/${id}`, changes));
    this.tasksResource.reload();
  }

  async toggle(task: Task) {
    // optimistic update: change UI immediately, then sync
    this.tasksResource.value.update(ts => ts.map(t => (t.id === task.id ? { ...t, done: !t.done } : t)));
    await firstValueFrom(this.http.patch(`/api/tasks/${task.id}`, { done: !task.done }));
  }

  async remove(id: string) {
    this.tasksResource.value.update(ts => ts.filter(t => t.id !== id));
    await firstValueFrom(this.http.delete(`/api/tasks/${id}`));
  }
}
```

## 6. Presentational component: `TaskItem`

```ts
// features/tasks/task-item.ts
import { Component, input, output } from '@angular/core';
import { RouterLink } from '@angular/router';
import { Task } from './task.model';
import { RelativeDuePipe } from '../../shared/relative-due.pipe';

@Component({
  selector: 'app-task-item',
  imports: [RouterLink, RelativeDuePipe],
  template: `
    <input type="checkbox" [checked]="task().done" (change)="toggle.emit(task())" [attr.aria-label]="'Toggle ' + task().title" />
    <div class="body">
      <span class="title">{{ task().title }}</span>
      <small>{{ task().dueDate | relativeDue }}</small>
    </div>
    @switch (task().priority) {
      @case ('high') { <span class="badge high">High</span> }
      @case ('medium') { <span class="badge medium">Medium</span> }
      @case ('low') { <span class="badge low">Low</span> }
    }
    <a [routerLink]="[task().id, 'edit']">Edit</a>
    <button (click)="remove.emit(task().id)" aria-label="Delete">✕</button>
  `,
  host: { '[class.done]': 'task().done' },
  styles: `
    :host { display: flex; gap: .75rem; align-items: center; padding: .5rem 0; border-bottom: 1px solid #eee; }
    :host(.done) .title { text-decoration: line-through; opacity: .6; }
    .body { flex: 1; display: flex; flex-direction: column; }
    .badge { font-size: .75rem; padding: .1rem .4rem; border-radius: 4px; }
    .high { background: #fdd; } .medium { background: #ffd; } .low { background: #dfd; }
  `,
})
export class TaskItem {
  task = input.required<Task>();
  toggle = output<Task>();
  remove = output<string>();
}
```

## 7. Stats widget (deferred)

```ts
// features/tasks/task-stats.ts
import { Component, inject } from '@angular/core';
import { TaskStore } from './task.store';

@Component({
  selector: 'app-task-stats',
  template: `
    @let s = store.stats();
    <section>
      <progress [value]="s.pct" max="100"></progress>
      {{ s.done }} / {{ s.total }} done ({{ s.pct }}%) · {{ s.open }} open
    </section>
  `,
})
export class TaskStats {
  protected store = inject(TaskStore);
}
```

## 8. Smart component: `TaskListPage`

```ts
// features/tasks/task-list-page.ts
import { Component, inject } from '@angular/core';
import { RouterLink } from '@angular/router';
import { Filter, TaskStore } from './task.store';
import { TaskItem } from './task-item';
import { TaskStats } from './task-stats';

@Component({
  selector: 'app-task-list-page',
  imports: [RouterLink, TaskItem, TaskStats],
  template: `
    <h1>Tasks</h1>

    <div class="toolbar">
      <input placeholder="Search…" [value]="store.search()" (input)="store.search.set($any($event.target).value)" />
      @for (f of filters; track f) {
        <button [class.active]="store.filter() === f" (click)="store.filter.set(f)">{{ f }}</button>
      }
    </div>

    @if (store.error()) {
      <p class="error">Couldn't load tasks. Is json-server running?</p>
    } @else if (store.loading() && store.tasks().length === 0) {
      <p>Loading…</p>
    } @else {
      @for (task of store.visible(); track task.id) {
        <app-task-item [task]="task" (toggle)="store.toggle($event)" (remove)="confirmRemove($event)" />
      } @empty {
        <p>No tasks. <a routerLink="new">Create one</a></p>
      }
    }

    @defer (on idle) {
      <app-task-stats />
    } @placeholder {
      <p>…</p>
    }
  `,
  styles: `
    .toolbar { display: flex; gap: .5rem; margin-bottom: 1rem; }
    .active { font-weight: bold; text-decoration: underline; }
    .error { color: crimson; }
  `,
})
export class TaskListPage {
  protected store = inject(TaskStore);
  protected readonly filters: Filter[] = ['all', 'active', 'done'];

  confirmRemove(id: string) {
    if (confirm('Delete this task?')) this.store.remove(id);
  }
}
```

## 9. Create/edit form with Signal Forms: `TaskFormPage`

One component handles both `/tasks/new` and `/tasks/:id/edit`. The `id` route param arrives as an
input thanks to `withComponentInputBinding()`.

```ts
// features/tasks/task-form-page.ts
import { Component, computed, inject, input, linkedSignal, signal } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { Router } from '@angular/router';
import { form, FormField, required, maxLength } from '@angular/forms/signals';
import { Task, TaskDraft, emptyDraft } from './task.model';
import { TaskStore } from './task.store';
import { Autofocus } from '../../shared/autofocus.directive';

@Component({
  selector: 'app-task-form-page',
  imports: [FormField, Autofocus],
  template: `
    <h1>{{ isEdit() ? 'Edit task' : 'New task' }}</h1>

    @if (existing.isLoading()) {
      <p>Loading…</p>
    } @else {
      <form (submit)="save($event)">
        <label>
          Title
          <input appAutofocus [formField]="f.title" />
        </label>
        @if (showErrors(f.title().touched())) {
          @for (err of f.title().errors(); track err.kind) { <small class="error">{{ err.message }}</small> }
        }

        <label>
          Priority
          <select [formField]="f.priority">
            <option value="low">Low</option>
            <option value="medium">Medium</option>
            <option value="high">High</option>
          </select>
        </label>

        <label>
          Due date
          <input type="date" [formField]="f.dueDate" />
        </label>
        @if (showErrors(f.dueDate().touched())) {
          @for (err of f.dueDate().errors(); track err.kind) { <small class="error">{{ err.message }}</small> }
        }

        <label><input type="checkbox" [formField]="f.done" /> Done</label>

        <button type="submit" [disabled]="saving()">{{ saving() ? 'Saving…' : 'Save' }}</button>
        <button type="button" (click)="cancel()">Cancel</button>
      </form>
    }
  `,
  styles: `
    form { display: grid; gap: .75rem; max-width: 420px; }
    label { display: grid; gap: .25rem; }
    .error { color: crimson; }
  `,
})
export class TaskFormPage {
  private store = inject(TaskStore);
  private router = inject(Router);

  id = input<string>();                                    // from the :id route param
  protected isEdit = computed(() => !!this.id());

  // load the existing task only in edit mode (returning undefined skips the request)
  protected existing = httpResource<Task>(() => (this.id() ? `/api/tasks/${this.id()}` : undefined));

  // form model: resets whenever the loaded task changes, but stays editable
  protected model = linkedSignal<TaskDraft>(() => {
    const t = this.existing.value();
    return t ? { title: t.title, priority: t.priority, dueDate: t.dueDate, done: t.done } : emptyDraft();
  });

  protected f = form(this.model, path => {
    required(path.title, { message: 'Title is required' });
    maxLength(path.title, 80, { message: 'Max 80 characters' });
    required(path.dueDate, { message: 'Pick a due date' });
  });

  protected saving = signal(false);
  private submitted = signal(false);

  protected showErrors(touched: boolean) {
    return touched || this.submitted();
  }

  async save(event: Event) {
    event.preventDefault();                 // no FormsModule, so stop the native submit
    this.submitted.set(true);
    if (this.f().invalid()) return;

    this.saving.set(true);
    try {
      const id = this.id();
      id ? await this.store.update(id, this.model()) : await this.store.create(this.model());
      this.router.navigate(['/tasks']);
    } finally {
      this.saving.set(false);
    }
  }

  cancel() { this.router.navigate(['/tasks']); }
}
```

Things to notice:
- **`linkedSignal`** turns async-loaded data into editable local state. In React you'd do this with
  `useEffect(() => setForm(data), [data])` or a `key` reset.
- The form **is** the signal: `this.model()` always holds the current values, with no `getRawValue()`.
- One component serves both routes because `id` is an optional input.

## 10. Login page

```ts
// login/login-page.ts
import { Component, inject, input, signal } from '@angular/core';
import { Router } from '@angular/router';
import { form, FormField, required, minLength } from '@angular/forms/signals';
import { AuthStore } from '../core/auth.store';
import { Autofocus } from '../shared/autofocus.directive';

@Component({
  selector: 'app-login-page',
  imports: [FormField, Autofocus],
  template: `
    <h1>Log in</h1>
    <form (submit)="login($event)">
      <input appAutofocus placeholder="Your name" [formField]="f.name" />
      <button type="submit" [disabled]="f().invalid()">Enter</button>
    </form>
  `,
})
export class LoginPage {
  private auth = inject(AuthStore);
  private router = inject(Router);
  returnUrl = input('/tasks');                       // from ?returnUrl= query param

  protected model = signal({ name: '' });
  protected f = form(this.model, p => {
    required(p.name);
    minLength(p.name, 2);
  });

  login(e: Event) {
    e.preventDefault();
    if (this.f().invalid()) return;
    this.auth.login(this.model().name.trim());
    this.router.navigateByUrl(this.returnUrl());
  }
}
```

## 11. Tests

```ts
// shared/relative-due.pipe.spec.ts
import { RelativeDuePipe } from './relative-due.pipe';

describe('RelativeDuePipe', () => {
  const pipe = new RelativeDuePipe();
  const today = new Date(2026, 9, 2);   // Oct 2, 2026

  it.each([
    ['2026-10-02', 'Due today'],
    ['2026-10-03', 'Due tomorrow'],
    ['2026-10-07', 'Due in 5d'],
    ['2026-09-30', 'Overdue by 2d'],
    [null, 'No due date'],
  ])('%s → %s', (date, expected) => {
    expect(pipe.transform(date, today)).toBe(expected);
  });
});
```

```ts
// features/tasks/task-item.spec.ts
import { TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { TaskItem } from './task-item';
import { Task } from './task.model';

describe('TaskItem', () => {
  const task: Task = { id: '1', title: 'Write tests', priority: 'high', dueDate: '2026-10-05', done: false };

  async function setup(t = task) {
    TestBed.configureTestingModule({ imports: [TaskItem], providers: [provideRouter([])] });
    const fixture = TestBed.createComponent(TaskItem);
    fixture.componentRef.setInput('task', t);
    await fixture.whenStable();
    return { fixture, el: fixture.nativeElement as HTMLElement };
  }

  it('renders title and priority badge', async () => {
    const { el } = await setup();
    expect(el.textContent).toContain('Write tests');
    expect(el.querySelector('.badge.high')).toBeTruthy();
  });

  it('emits toggle with the task', async () => {
    const { fixture, el } = await setup();
    const spy = vi.fn();
    fixture.componentInstance.toggle.subscribe(spy);
    el.querySelector<HTMLInputElement>('input[type=checkbox]')!.click();
    expect(spy).toHaveBeenCalledWith(task);
  });

  it('adds the done class on the host', async () => {
    const { el } = await setup({ ...task, done: true });
    expect(el.classList).toContain('done');
  });
});
```
Run them with `ng test`.

## 12. Stretch goals (practice the remaining chapters)

1. **Unsaved-changes guard**: `canDeactivate` on the form page using `f().dirty()`.
2. **Debounced server search**: replace client filtering with `toObservable(search)` +
   `debounceTime` + `switchMap` (chapter 09), or a debounced `httpResource` URL.
3. **NgRx SignalStore**: rewrite `TaskStore` with `signalStore` (chapter 14).
4. **Role directive**: `*appHasRole="'admin'"` to hide Delete for non-admins (chapter 06).
5. **Error interceptor + toast service** (chapter 10).
6. **`@boundary`** around `<app-task-stats />` (v22.2 preview) with a fallback.
7. **Reactive Forms version** of the task form, to practice what most existing codebases use (chapter 12).
8. **SSR**: `ng add @angular/ssr`, prerender `/login`, then fix the `localStorage` access in `AuthStore`
   for the server (chapter 17).
9. **Drag & drop reordering** with `@angular/cdk/drag-drop`.
10. **Playwright E2E**: log in → create task → toggle → delete.
