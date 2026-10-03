# 15 — Testing

| | AngularJS 1.x | React | Angular |
|---|---|---|---|
| Runner | Karma + Jasmine | Jest / Vitest | **Vitest** (default for new projects since v21); Jasmine + Karma in older projects (Karma deprecated); Jest via community |
| Component testing | `$compile` + `$rootScope.$digest()` | React Testing Library | `TestBed` + `ComponentFixture`, or **Angular Testing Library** |
| HTTP mocking | `$httpBackend` | msw | `HttpTestingController` |
| E2E | Protractor (dead) | Cypress / Playwright | Cypress / Playwright |

```bash
ng test            # unit tests
ng e2e             # after adding an e2e package (ng add @playwright/test style schematics / cypress)
```

## 1. Testing a service (plain class + DI)

```ts
// cart.store.spec.ts
import { TestBed } from '@angular/core/testing';
import { CartStore } from './cart.store';

describe('CartStore', () => {
  let store: CartStore;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    store = TestBed.inject(CartStore);
  });

  it('adds items and computes totals', () => {
    store.add({ id: 1, name: 'Pen', price: 2 });
    store.add({ id: 1, name: 'Pen', price: 2 });
    expect(store.count()).toBe(2);
    expect(store.total()).toBe(4);
  });
});
```
A service without dependencies can also be tested with `new CartStore()`, but `inject()` calls
need an injection context, so use `TestBed.inject` or `TestBed.runInInjectionContext`.

## 2. Testing a component

```ts
// user-card.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserCard } from './user-card';

describe('UserCard', () => {
  let fixture: ComponentFixture<UserCard>;
  let el: HTMLElement;

  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [UserCard] }).compileComponents();
    fixture = TestBed.createComponent(UserCard);
    fixture.componentRef.setInput('user', { id: 1, name: 'Ana Lima', email: 'a@x.com', joined: new Date() });
    await fixture.whenStable();          // zoneless-friendly; older code: fixture.detectChanges()
    el = fixture.nativeElement;
  });

  it('renders the name and initials', () => {
    expect(el.querySelector('h3')?.textContent).toContain('ANA LIMA');
    expect(el.querySelector('.avatar')?.textContent).toBe('AL');
  });

  it('emits select with the user id', () => {
    const spy = vi.fn();                              // jasmine.createSpy() in Jasmine
    fixture.componentInstance.select.subscribe(spy);  // outputs support subscribe()
    el.querySelector('button')!.click();
    expect(spy).toHaveBeenCalledWith(1);
  });

  it('updates when input changes', async () => {
    fixture.componentRef.setInput('user', { id: 2, name: 'Bo Chen', email: '', joined: new Date() });
    await fixture.whenStable();
    expect(el.querySelector('.avatar')?.textContent).toBe('BC');
  });
});
```
Key APIs: `TestBed.createComponent`, `fixture.componentInstance`, `fixture.nativeElement`,
`fixture.debugElement.query(By.css('...'))`, `fixture.componentRef.setInput()`,
`fixture.detectChanges()`, `await fixture.whenStable()`.

## 3. Mocking dependencies

```ts
const userApiMock = { list: vi.fn(() => of([{ id: 1, name: 'Ana' }])) };

await TestBed.configureTestingModule({
  imports: [UserList],
  providers: [
    { provide: UserApi, useValue: userApiMock },
    provideRouter([]),                 // if the component uses routerLink
  ],
}).compileComponents();
```
Override a component-level provider:
```ts
TestBed.overrideComponent(CheckoutWizard, {
  set: { providers: [{ provide: WizardState, useValue: fakeState }] },
});
```

## 4. Angular Testing Library (RTL style, recommended for React devs)

```ts
import { render, screen } from '@testing-library/angular';
import userEvent from '@testing-library/user-event';

it('adds a todo', async () => {
  await render(Todos);
  await userEvent.type(screen.getByPlaceholderText('What needs doing?'), 'Buy milk{enter}');
  expect(screen.getByText('Buy milk')).toBeTruthy();
  expect(screen.getByText('1 left')).toBeTruthy();
});

it('renders with inputs', async () => {
  await render(UserCard, { inputs: { user: mockUser } });
  expect(screen.getByRole('heading')).toHaveTextContent('ANA');
});
```
Same queries, same philosophy: test what the user sees.

## 5. Testing HTTP
See [chapter 10](./10-http-client.md#6-testing-http): `provideHttpClient()`,
`provideHttpClientTesting()`, `HttpTestingController.expectOne().flush()`, `verify()`.

## 6. Testing async code

```ts
// Vitest fake timers
vi.useFakeTimers();
component.search('ang');
vi.advanceTimersByTime(300);         // debounceTime(300)
await fixture.whenStable();

// zone.js-based projects (Jasmine) use fakeAsync/tick
it('debounces', fakeAsync(() => {
  component.search('ang');
  tick(300);
  fixture.detectChanges();
  expect(...).toBe(...);
}));
```

## 7. Testing signals, effects, guards

```ts
// effects run during change detection — flush them explicitly
TestBed.tick();     // runs pending effects / CD (older: TestBed.flushEffects())

// functional guard
it('redirects anonymous users', () => {
  TestBed.configureTestingModule({ providers: [{ provide: AuthStore, useValue: { isLoggedIn: () => false } }, provideRouter([])] });
  const result = TestBed.runInInjectionContext(() => authGuard({} as any, { url: '/users' } as any));
  expect(result).toBeInstanceOf(UrlTree);
});

// routed component with real router
const harness = await RouterTestingHarness.create();
const page = await harness.navigateByUrl('/users/1', UserDetail);
```

## 8. Component harnesses (Angular CDK/Material)

```ts
const loader = TestbedHarnessEnvironment.loader(fixture);
const button = await loader.getHarness(MatButtonHarness.with({ text: 'Save' }));
await button.click();
```
Harnesses give a stable API over component internals (like page objects).

## 9. Interview questions

1. **What is TestBed?** Angular's testing utility that configures a test injector/environment and
   creates components with their dependencies.
2. **How do you set inputs in tests?** `fixture.componentRef.setInput('name', value)`, which works for
   both `input()` and `@Input()`.
3. **`detectChanges` vs `whenStable`?** `detectChanges` runs CD synchronously; `whenStable` waits
   until pending async work and CD settle (preferred for zoneless).
4. **`fakeAsync`/`tick`?** zone.js-based virtual time control for timers and promises. Zoneless/Vitest
   setups use the runner's fake timers.
5. **How do you mock a service?** Provide a replacement: `{ provide: Service, useValue: mock }`.
6. **What are component harnesses?** Test APIs that interact with components the way a user
   would, insulating tests from DOM structure changes.
