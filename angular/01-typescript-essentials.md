# 01 — TypeScript Essentials for Angular

Angular is written in and for TypeScript. If you've used React + TS, most of this is familiar;
focus on the parts marked **(Angular-heavy)** — classes, access modifiers and decorators, which
React code rarely uses.

---

## 1. Basic and structural types

```ts
let id: number = 1;
let name: string = 'Ana';
let active: boolean = true;
let tags: string[] = ['a', 'b'];
let pair: [string, number] = ['age', 30];     // tuple
let anything: unknown;                         // prefer unknown over any

interface User {
  id: number;
  name: string;
  email?: string;                // optional
  readonly createdAt: Date;      // can't be reassigned
}

type Status = 'idle' | 'loading' | 'success' | 'error';   // union of literals (prefer over enum)
type ApiResult<T> = { data: T; error: null } | { data: null; error: string };
```

`interface` vs `type`: both describe object shapes. Interfaces can be extended/merged; types can
express unions, intersections and mapped types. Angular code commonly uses `interface` for models.

## 2. Classes (Angular-heavy)

Components, services, directives, pipes and guards are often classes, so you need class syntax.

```ts
class Animal {
  // access modifiers
  public name: string;          // default
  protected sound = '...';      // subclasses can see it
  private secret = 42;          // only this class (compile-time)
  #reallyPrivate = 1;           // JS private field (runtime)

  static count = 0;             // on the class, not instances

  constructor(name: string) {
    this.name = name;
    Animal.count++;
  }

  speak(): string {
    return `${this.name} says ${this.sound}`;
  }
}

class Dog extends Animal {
  protected override sound = 'woof';
}
```

### Parameter properties (you'll see this in all legacy Angular code)
```ts
// Long form
class UserService {
  private http: HttpClient;
  constructor(http: HttpClient) { this.http = http; }
}

// Short form — same thing. This is classic Angular constructor injection.
class UserService {
  constructor(private http: HttpClient) {}
}

// Modern Angular prefers the inject() function instead:
class UserService {
  private http = inject(HttpClient);
}
```

### `protected` / `private` in components
Since v15 templates can access `protected` members, and since **v22.2 also `private`** members.
The common convention (which you'll see in most codebases): members only used in the template are
`protected`, and the public API (inputs/outputs/methods) is `public`.

### Definite assignment `!`
```ts
class Foo {
  value!: string;   // "trust me, it'll be set before use" — common with legacy @Input / @ViewChild
}
```

## 3. Decorators (Angular-heavy)

A decorator is a function that attaches metadata to a class, property or parameter. Angular
reads that metadata at compile time.

```ts
@Component({ selector: 'app-hello', template: '<p>Hello</p>' })   // class decorator
export class Hello {
  @Input() name = '';                 // property decorator (legacy input)
  @Output() clicked = new EventEmitter<void>();
  @HostListener('click') onClick() {} // method decorator
  constructor(@Inject(API_URL) private url: string) {} // parameter decorator
}
```

Class decorators you'll use: `@Component`, `@Directive`, `@Pipe`, `@Injectable`, `@NgModule`.
Member decorators (`@Input`, `@Output`, `@ViewChild`, `@HostListener`, `@HostBinding`) are being
replaced by functions: `input()`, `output()`, `viewChild()` and the `host` metadata object.

## 4. Generics

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}

class Store<T> {
  private state: T;
  constructor(initial: T) { this.state = initial; }
  get(): T { return this.state; }
  set(next: T) { this.state = next; }
}

// Angular APIs are generic everywhere:
signal<User | null>(null);
http.get<User[]>('/api/users');
input.required<User>();
output<number>();
new InjectionToken<string>('API_URL');
```

## 5. Narrowing and null safety

Angular projects use `strict: true`. Expect to handle `null`/`undefined` explicitly.

```ts
function label(user: User | null) {
  if (!user) return 'Guest';          // narrowing
  return user.email ?? user.name;      // nullish coalescing
}

const city = user?.address?.city;      // optional chaining (works in templates too)

function isError(r: ApiResult<unknown>): r is { data: null; error: string } {  // type guard
  return r.error !== null;
}
```

Strict template type checking (`strictTemplates`) means templates are type-checked too — a typo
in `{{ user.nmae }}` fails the build.

## 6. Utility types you'll use a lot

```ts
Partial<User>             // all optional — great for PATCH payloads / form values
Required<User>
Readonly<User>
Pick<User, 'id' | 'name'>
Omit<User, 'createdAt'>   // e.g. create-user DTO
Record<string, number>    // dictionary
ReturnType<typeof fn>
Awaited<Promise<User>>    // User
```

## 7. Modules (ES modules)

```ts
// user.model.ts
export interface User { id: number; name: string }
export const DEFAULT_USER: User = { id: 0, name: 'Guest' };

// elsewhere
import { User, DEFAULT_USER } from './user.model';
```

Don't confuse **ES modules** (files with import/export) with **NgModules** (`@NgModule` classes
that group Angular declarations). Modern Angular mostly uses ES modules alone.

## 8. Enums vs union types

```ts
enum Role { Admin = 'ADMIN', User = 'USER' }   // generates runtime JS object
type Role2 = 'ADMIN' | 'USER';                  // zero runtime cost — usually preferred
```

## 9. Common interview questions

1. **`interface` vs `type`?** Interfaces are extendable and can merge declarations; types support
   unions, intersections and mapped/conditional types. Either works for object shapes.
2. **What is a decorator?** A function applied with `@` that adds metadata or behavior to a
   class/member. Angular uses it to know that a class is a component, service, etc.
3. **`unknown` vs `any`?** `unknown` forces you to narrow before use; `any` disables type checking.
4. **What does `private` in a constructor parameter do?** It declares and assigns a class property
   in one step (parameter property).
5. **Why does Angular care about `strict` mode?** Catches null bugs at compile time and enables
   strict template type checking.
