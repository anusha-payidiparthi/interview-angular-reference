# 10 — HTTP Client

| | AngularJS 1.x | React | Angular |
|---|---|---|---|
| Client | `$http` (Promise) | `fetch` / axios | `HttpClient` (Observable) |
| Interceptors | `$httpProvider.interceptors` | axios interceptors | `HttpInterceptorFn` |
| Caching/loading state | manual | React Query / SWR | manual, `httpResource`, or NgRx |
| Testing | `$httpBackend` | msw / jest mocks | `HttpTestingController` |

## 1. Setup

```ts
// app.config.ts
import { provideHttpClient, withFetch, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './core/auth.interceptor';
import { errorInterceptor } from './core/error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withFetch(),                                         // use fetch API (recommended, needed for SSR streaming)
      withInterceptors([authInterceptor, errorInterceptor]),
    ),
  ],
};
```
Legacy: `imports: [HttpClientModule]` in `AppModule`.

## 2. A typed API service

```ts
// user-api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface User { id: number; name: string; email: string }
export type CreateUser = Omit<User, 'id'>;

@Injectable({ providedIn: 'root' })
export class UserApi {
  private http = inject(HttpClient);
  private base = '/api/users';

  list(page = 1, search = ''): Observable<User[]> {
    const params = new HttpParams().set('page', page).set('q', search);
    return this.http.get<User[]>(this.base, { params });
  }

  get(id: number) { return this.http.get<User>(`${this.base}/${id}`); }
  create(body: CreateUser) { return this.http.post<User>(this.base, body); }
  update(id: number, body: Partial<User>) { return this.http.patch<User>(`${this.base}/${id}`, body); }
  remove(id: number) { return this.http.delete<void>(`${this.base}/${id}`); }

  // full response (headers, status)
  listWithTotal(page: number) {
    return this.http.get<User[]>(this.base, { params: { page }, observe: 'response' });
    // → Observable<HttpResponse<User[]>>, read res.headers.get('X-Total-Count')
  }

  // upload progress
  upload(file: File) {
    const form = new FormData();
    form.append('file', file);
    return this.http.post('/api/upload', form, { reportProgress: true, observe: 'events' });
  }
}
```
> The generic `get<User>` is a **type assertion**, not validation. Validate with zod or similar if
> the API is untrusted.

## 3. Using it in a component

### Option A: signals via `toSignal` (simple read)
```ts
export class UserList {
  private api = inject(UserApi);
  users = toSignal(this.api.list(), { initialValue: [] });
}
```

### Option B: explicit loading/error state (most common in real apps)
```ts
export class UserList {
  private api = inject(UserApi);
  private destroyRef = inject(DestroyRef);

  users = signal<User[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);

  ngOnInit() { this.load(); }

  load() {
    this.loading.set(true);
    this.error.set(null);
    this.api.list()
      .pipe(
        finalize(() => this.loading.set(false)),
        takeUntilDestroyed(this.destroyRef),      // outside constructor → pass DestroyRef
      )
      .subscribe({
        next: users => this.users.set(users),
        error: () => this.error.set('Failed to load users'),
      });
  }

  delete(id: number) {
    this.api.remove(id).subscribe(() =>
      this.users.update(list => list.filter(u => u.id !== id))   // optimistic-ish update
    );
  }
}
```

### Option C: `httpResource` (stable since v22; experimental in v19.2–v21) — closest to React Query

This is the recommended way to **read** data in v22 apps.
```ts
import { httpResource } from '@angular/common/http';

export class UserDetail {
  id = input.required<number>();
  user = httpResource<User>(() => `/api/users/${this.id()}`);   // refetches when id changes
}
```
```html
@if (user.isLoading()) { <app-spinner /> }
@if (user.error()) { <p>Error!</p> }
@if (user.hasValue()) { <h1>{{ user.value().name }}</h1> }
```
`httpResource` uses `HttpClient` under the hood, so interceptors still run. It's meant for
**reading** data; use `HttpClient` methods directly for mutations.

More options:
```ts
// request object instead of URL string; returning undefined skips the request
results = httpResource<User[]>(() =>
  this.query() ? { url: '/api/users', params: { q: this.query() } } : undefined,
  { defaultValue: [] },             // value() is never undefined
);

// after a mutation, refetch:
this.results.reload();
// or update locally (value is a WritableSignal) for optimistic UI:
this.results.value.update(list => [...list, newUser]);
```

### Option D: async pipe
```ts
users$ = this.api.list();
```
```html
@if (users$ | async; as users) { @for (u of users; track u.id) { ... } } @else { Loading… }
```

## 4. Functional interceptors

Interceptors run for every request: auth headers, logging, error handling, retries, caching,
loading spinners.

```ts
// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthStore).token();
  if (!token || !req.url.startsWith('/api')) return next(req);

  // requests are immutable → clone
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};
```

```ts
// error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const toast = inject(ToastService);

  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) router.navigate(['/login']);
      else if (err.status >= 500) toast.error('Server error, try again later');
      return throwError(() => err);       // let callers handle it too
    }),
  );
};
```

```ts
// loading.interceptor.ts — global spinner
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loader = inject(LoadingService);
  loader.start();
  return next(req).pipe(finalize(() => loader.stop()));
};
```
Order matters: requests flow through interceptors in array order, responses in reverse.

### Passing per-request options to interceptors: `HttpContext`
```ts
export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
this.http.get('/api/public', { context: new HttpContext().set(SKIP_AUTH, true) });
// in interceptor: if (req.context.get(SKIP_AUTH)) return next(req);
```

### Legacy class interceptor
```ts
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) { return next.handle(req.clone({...})); }
}
// providers: [{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }]
```

## 5. Environments & base URLs

```ts
// src/environments/environment.ts (generate with: ng g environments)
export const environment = { production: false, apiUrl: 'http://localhost:3000' };
```
Combine with an `InjectionToken` (chapter 07) or a base-URL interceptor. For local dev, a proxy
avoids CORS:
```json
// proxy.conf.json  → ng serve --proxy-config proxy.conf.json
{ "/api": { "target": "http://localhost:3000", "secure": false } }
```

## 6. Testing HTTP

```ts
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';

describe('UserApi', () => {
  let api: UserApi;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    api = TestBed.inject(UserApi);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());   // no unexpected requests

  it('fetches users', () => {
    let result: User[] = [];
    api.list().subscribe(u => (result = u));

    const req = http.expectOne(r => r.url === '/api/users');
    expect(req.request.method).toBe('GET');
    req.flush([{ id: 1, name: 'Ana', email: 'a@x.com' }]);

    expect(result.length).toBe(1);
  });
});
```

## 7. Gotchas

1. **No subscribe → no request.** Observables are lazy.
2. **Multiple `async` pipes on the same observable → multiple requests.** Use `shareReplay`, `toSignal`, or `@if (x$ | async; as x)` once.
3. `HttpErrorResponse.status === 0` means a network/CORS error.
4. `HttpClient` parses JSON by default. Use `{ responseType: 'text' | 'blob' }` for other formats.
5. Don't nest `subscribe` calls. Use `switchMap`/`forkJoin`.

## 8. Interview questions

1. **Why does HttpClient return Observables?** Cancellation (switchMap/unsubscribe), composability
   with operators, retries, progress events, and consistency with the rest of Angular.
2. **What is an interceptor? Give examples.** Middleware for HTTP requests/responses: auth
   headers, error handling, logging, caching, loading indicators.
3. **Why clone the request in an interceptor?** `HttpRequest` is immutable.
4. **How do you cancel an HTTP request?** Unsubscribe, or use `switchMap`/`takeUntil`.
5. **How do you test HTTP calls?** `provideHttpClientTesting()` + `HttpTestingController`
   (`expectOne`, `flush`, `verify`).
6. **What is `httpResource`?** A signal-based wrapper around HttpClient for reactive data fetching
   with built-in loading/error/value state.
