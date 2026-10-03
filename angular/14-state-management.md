# 14 — State Management

## 1. The ladder (use the lowest rung that works)

| Level | Angular | React equivalent | AngularJS 1.x |
|---|---|---|---|
| 1. Local UI state | `signal()` in the component | `useState` | controller properties |
| 2. Shared between parent/children | inputs/outputs/`model()` | props / lifting state | bindings |
| 3. Shared across a subtree | Service in component `providers` | Context Provider | — |
| 4. App-wide state | **Service with signals** (`@Service()` / `providedIn: 'root'`) | Context + hooks / Zustand | services + `$rootScope` |
| 5. Structured store, lighter | **NgRx SignalStore** | Zustand / Redux Toolkit slices | — |
| 6. Full Redux pattern | **NgRx Store** (+ Effects, Entity, DevTools) | Redux + Redux-Saga/Thunk | — |
| Server cache | `httpResource`, or TanStack Query for Angular | React Query | — |

In practice: **most modern Angular apps get far with signal-based services**. Large enterprise
codebases often use **NgRx Store**, so you need to be able to read it.

## 2. Signal-based service store (the default choice)

```ts
// todo.store.ts
import { Injectable, computed, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';

export interface Todo { id: number; title: string; done: boolean }
type Filter = 'all' | 'active' | 'done';

interface TodoState {
  todos: Todo[];
  filter: Filter;
  loading: boolean;
  error: string | null;
}

@Injectable({ providedIn: 'root' })        // v22+: @Service() is equivalent
export class TodoStore {
  private http = inject(HttpClient);

  private readonly state = signal<TodoState>({ todos: [], filter: 'all', loading: false, error: null });

  // selectors
  readonly todos = computed(() => this.state().todos);
  readonly filter = computed(() => this.state().filter);
  readonly loading = computed(() => this.state().loading);
  readonly error = computed(() => this.state().error);
  readonly visible = computed(() => {
    const { todos, filter } = this.state();
    return filter === 'all' ? todos : todos.filter(t => (filter === 'done' ? t.done : !t.done));
  });
  readonly remaining = computed(() => this.todos().filter(t => !t.done).length);

  // actions
  setFilter(filter: Filter) { this.patch({ filter }); }

  async load() {
    this.patch({ loading: true, error: null });
    try {
      const todos = await firstValueFrom(this.http.get<Todo[]>('/api/todos'));
      this.patch({ todos, loading: false });
    } catch {
      this.patch({ loading: false, error: 'Failed to load todos' });
    }
  }

  toggle(id: number) {
    this.patch({ todos: this.todos().map(t => (t.id === id ? { ...t, done: !t.done } : t)) });
  }

  private patch(partial: Partial<TodoState>) {
    this.state.update(s => ({ ...s, ...partial }));
  }
}
```
```ts
// any component
export class TodoPage {
  protected store = inject(TodoStore);
  constructor() { this.store.load(); }
}
```
```html
@if (store.loading()) { <app-spinner /> }
@for (t of store.visible(); track t.id) { <app-todo-item [todo]="t" (toggle)="store.toggle($event)" /> }
<p>{{ store.remaining() }} remaining</p>
```

Compare to a Zustand store: `create(set => ({ todos: [], toggle: id => set(...) }))`. It's the
same shape, with DI instead of a module singleton and signals instead of selectors with equality
functions.

## 3. Classic RxJS service store (very common in existing code)

```ts
@Injectable({ providedIn: 'root' })
export class TodoStore {
  private readonly state$ = new BehaviorSubject<TodoState>(initialState);
  readonly todos$ = this.state$.pipe(map(s => s.todos), distinctUntilChanged());
  readonly remaining$ = this.todos$.pipe(map(ts => ts.filter(t => !t.done).length));

  toggle(id: number) {
    const s = this.state$.value;
    this.state$.next({ ...s, todos: s.todos.map(t => (t.id === id ? { ...t, done: !t.done } : t)) });
  }
}
// template: {{ store.remaining$ | async }}
```

## 4. NgRx SignalStore (`@ngrx/signals`)

A functional, composable store built on signals. It's popular for new projects that want structure
without Redux boilerplate.

```ts
import { signalStore, withState, withComputed, withMethods, patchState, withHooks } from '@ngrx/signals';
import { rxMethod } from '@ngrx/signals/rxjs-interop';

export const TodoStore = signalStore(
  { providedIn: 'root' },
  withState<TodoState>({ todos: [], filter: 'all', loading: false, error: null }),

  withComputed(({ todos, filter }) => ({
    visible: computed(() => filter() === 'all' ? todos() : todos().filter(t => (filter() === 'done') === t.done)),
    remaining: computed(() => todos().filter(t => !t.done).length),
  })),

  withMethods((store, api = inject(TodoApi)) => ({
    setFilter(filter: Filter) { patchState(store, { filter }); },
    toggle(id: number) {
      patchState(store, s => ({ todos: s.todos.map(t => (t.id === id ? { ...t, done: !t.done } : t)) }));
    },
    // RxJS-powered method: handles cancellation with switchMap
    load: rxMethod<void>(pipe(
      tap(() => patchState(store, { loading: true })),
      switchMap(() => api.list().pipe(
        tapResponse({
          next: todos => patchState(store, { todos, loading: false }),
          error: () => patchState(store, { loading: false, error: 'Failed' }),
        }),
      )),
    )),
  })),

  withHooks({ onInit(store) { store.load(); } }),
);

// usage: store = inject(TodoStore); store.visible(); store.toggle(1);
```
(`tapResponse` comes from `@ngrx/operators`.) There's also `withEntities()` for normalized
collections.

## 5. NgRx Store (Redux pattern)

If you know Redux Toolkit, this maps one-to-one:

| Redux | NgRx |
|---|---|
| action creators | `createAction` / `createActionGroup` |
| reducer | `createReducer` + `on()` |
| selectors (reselect) | `createSelector` / `createFeatureSelector` |
| thunks / sagas | **Effects** (`createEffect` + RxJS) |
| `useSelector` | `store.select()` / `store.selectSignal()` |
| `dispatch` | `store.dispatch()` |
| `createEntityAdapter` | `@ngrx/entity` |
| Redux DevTools | `@ngrx/store-devtools` |

```ts
// todo.actions.ts
export const TodoActions = createActionGroup({
  source: 'Todos',
  events: {
    'Load': emptyProps(),
    'Load Success': props<{ todos: Todo[] }>(),
    'Load Failure': props<{ error: string }>(),
    'Toggle': props<{ id: number }>(),
  },
});

// todo.reducer.ts
export const todoFeature = createFeature({
  name: 'todos',
  reducer: createReducer(
    initialState,
    on(TodoActions.load, s => ({ ...s, loading: true })),
    on(TodoActions.loadSuccess, (s, { todos }) => ({ ...s, todos, loading: false })),
    on(TodoActions.loadFailure, (s, { error }) => ({ ...s, error, loading: false })),
    on(TodoActions.toggle, (s, { id }) => ({ ...s, todos: s.todos.map(t => (t.id === id ? { ...t, done: !t.done } : t)) })),
  ),
  extraSelectors: ({ selectTodos }) => ({
    selectRemaining: createSelector(selectTodos, todos => todos.filter(t => !t.done).length),
  }),
});

// todo.effects.ts — side effects (API calls) live here, not in reducers
export const loadTodos = createEffect(
  (actions$ = inject(Actions), api = inject(TodoApi)) =>
    actions$.pipe(
      ofType(TodoActions.load),
      switchMap(() => api.list().pipe(
        map(todos => TodoActions.loadSuccess({ todos })),
        catchError(() => of(TodoActions.loadFailure({ error: 'Failed' }))),
      )),
    ),
  { functional: true },
);

// app.config.ts
providers: [
  provideStore({ [todoFeature.name]: todoFeature.reducer }),
  provideEffects({ loadTodos }),
  provideStoreDevtools({ maxAge: 25 }),
]

// component
export class TodoPage {
  private store = inject(Store);
  todos = this.store.selectSignal(todoFeature.selectTodos);
  remaining = this.store.selectSignal(todoFeature.selectRemaining);
  constructor() { this.store.dispatch(TodoActions.load()); }
  toggle(id: number) { this.store.dispatch(TodoActions.toggle({ id })); }
}
```

## 6. Choosing (an interview-ready answer)

- **Signals in components/services**: the default. Low ceremony, enough for most apps.
- **SignalStore**: when several teams need consistent store structure, plugins (entities, devtools),
  and `rxMethod` for async.
- **NgRx Store**: large apps with complex cross-feature state, strict unidirectional flow, time-travel
  debugging, and an action log for auditing. Costs boilerplate.
- **Server state** (caching, refetch, dedupe): `httpResource`, or TanStack Query Angular. Don't put
  everything in a global store.

## 7. Interview questions

1. **How do you manage state in Angular?** Walk the ladder in section 1.
2. **Why NgRx? Downsides?** Predictability, devtools, separation of side effects, scalability. The
   downsides are boilerplate, the learning curve (RxJS), and indirection.
3. **What are NgRx effects?** Listeners on the action stream that perform side effects (HTTP) and
   dispatch new actions. They keep reducers pure.
4. **What is a selector? Why memoized?** A pure function to derive state. Memoization avoids
   recomputation and needless re-renders.
5. **Signals vs BehaviorSubject for service state?** Signals are synchronous, glitch-free, need no
   subscription management, and integrate with CD. BehaviorSubject is better when you need RxJS
   operators or stream semantics.
6. **What is SignalStore?** NgRx's signal-based, functional store: `withState`, `withComputed`,
   `withMethods`, `withHooks`, and extensible features.
