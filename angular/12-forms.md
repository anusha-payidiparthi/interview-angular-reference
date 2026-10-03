# 12 — Forms

Angular ships **three form systems**:

| | Template-driven | Reactive | **Signal Forms (stable in v22)** |
|---|---|---|---|
| Source of truth | Template (`ngModel`) | Class (`FormGroup`/`FormControl`) | A signal holding the model |
| Import | `FormsModule` | `ReactiveFormsModule` | `FormField` from `@angular/forms/signals` |
| AngularJS analogy | `ng-model` + `name` + `$valid`, very similar | — | — |
| React analogy | uncontrolled-ish / simple controlled inputs | **React Hook Form / Formik** | Controlled state + zod-like schema |
| Best for | Small/simple forms | **Most existing company code** (industry default up to v21) | **New code in v22+** |
| Typing | Weak | Strongly typed (v14+) | Strongly typed |

**What to learn:** Reactive Forms (you *will* meet them at work) **and** Signal Forms (the
recommended API for new forms in v22). Template-driven forms are easy once you know the other two.
Signal Forms are covered in [section 6](#6-signal-forms-stable-in-v22).

## 1. Template-driven forms (the AngularJS-like way)

```ts
import { Component } from '@angular/core';
import { FormsModule, NgForm } from '@angular/forms';

@Component({
  selector: 'app-contact',
  imports: [FormsModule],
  template: `
    <form #f="ngForm" (ngSubmit)="submit(f)">
      <input name="name" [(ngModel)]="model.name" required minlength="2" #name="ngModel" />
      @if (name.invalid && name.touched) {
        <small>Name is required (min 2 chars)</small>
      }

      <input name="email" type="email" [(ngModel)]="model.email" required email />

      <button [disabled]="f.invalid">Send</button>
    </form>
  `,
})
export class Contact {
  model = { name: '', email: '' };
  submit(f: NgForm) { console.log(f.value, this.model); f.resetForm(); }
}
```
If you remember AngularJS forms (`form.name.$invalid`, `$touched`), this is nearly identical.

## 2. Reactive forms (the one to master)

### Basic
```ts
import { Component, inject } from '@angular/core';
import { ReactiveFormsModule, NonNullableFormBuilder, Validators } from '@angular/forms';

@Component({
  selector: 'app-signup',
  imports: [ReactiveFormsModule, JsonPipe],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <label>Email <input formControlName="email" type="email" /></label>
      @if (form.controls.email.touched && form.controls.email.errors; as errs) {
        @if (errs['required']) { <small>Email is required</small> }
        @if (errs['email']) { <small>Invalid email</small> }
      }

      <label>Password <input formControlName="password" type="password" /></label>

      <div formGroupName="address">                         <!-- nested group -->
        <input formControlName="city" placeholder="City" />
        <input formControlName="zip" placeholder="ZIP" />
      </div>

      <label><input type="checkbox" formControlName="terms" /> Accept terms</label>

      <button type="submit" [disabled]="form.invalid || saving()">Sign up</button>
    </form>
    <pre>{{ form.value | json }}</pre>
  `,
})
export class Signup {
  private fb = inject(NonNullableFormBuilder);   // values reset to initial, never null
  saving = signal(false);

  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
    address: this.fb.group({
      city: [''],
      zip: ['', Validators.pattern(/^\d{5}$/)],
    }),
    terms: [false, Validators.requiredTrue],
  });

  submit() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();          // show all errors
      return;
    }
    const value = this.form.getRawValue();   // fully typed: { email: string; password: string; address: {...}; terms: boolean }
    console.log(value);
  }
}
```

### Without FormBuilder (explicit)
```ts
form = new FormGroup({
  email: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
  age: new FormControl<number | null>(null),
});
```

### Key API
```ts
form.value              // values of enabled controls
form.getRawValue()      // includes disabled controls
form.valid / invalid / pending / dirty / pristine / touched / untouched
form.controls.email     // typed access (or form.get('address.city'))
form.patchValue({ email: 'a@b.com' })   // partial update
form.setValue({...})                    // must provide all fields
form.reset()
control.disable() / enable()
control.setValidators([...]); control.updateValueAndValidity();
form.valueChanges / statusChanges       // Observables
```

### Reacting to changes
```ts
constructor() {
  // show/hide a field based on another
  this.form.controls.country.valueChanges.pipe(takeUntilDestroyed()).subscribe(country => {
    const state = this.form.controls.state;
    country === 'US' ? state.enable() : state.disable();
  });
}

// or as a signal
emailValue = toSignal(this.form.controls.email.valueChanges, { initialValue: '' });
```

## 3. Dynamic forms with `FormArray`

```ts
@Component({
  selector: 'app-invoice',
  imports: [ReactiveFormsModule, CurrencyPipe],
  template: `
    <form [formGroup]="form">
      <div formArrayName="lines">
        @for (line of lines.controls; track line; let i = $index) {
          <div [formGroupName]="i">
            <input formControlName="description" />
            <input formControlName="qty" type="number" />
            <input formControlName="price" type="number" />
            <button type="button" (click)="lines.removeAt(i)">✕</button>
          </div>
        }
      </div>
      <button type="button" (click)="addLine()">+ Line</button>
      <p>Total: {{ total() | currency }}</p>
    </form>
  `,
})
export class Invoice {
  private fb = inject(NonNullableFormBuilder);
  form = this.fb.group({ lines: this.fb.array([this.newLine()]) });
  get lines() { return this.form.controls.lines; }

  private formValue = toSignal(this.form.valueChanges, { initialValue: this.form.getRawValue() });
  total = computed(() =>
    (this.formValue().lines ?? []).reduce((s, l) => s + (l.qty ?? 0) * (l.price ?? 0), 0)
  );

  newLine() {
    return this.fb.group({ description: ['', Validators.required], qty: [1], price: [0] });
  }
  addLine() { this.lines.push(this.newLine()); }
}
```
React Hook Form equivalent: `useFieldArray`.

## 4. Custom validators

```ts
import { AbstractControl, ValidationErrors, ValidatorFn, AsyncValidatorFn } from '@angular/forms';

// sync, single control
export function forbiddenName(name: string): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null =>
    control.value?.toLowerCase() === name ? { forbiddenName: { value: control.value } } : null;
}

// cross-field (on the group)
export const passwordsMatch: ValidatorFn = group => {
  const { password, confirm } = group.value;
  return password === confirm ? null : { passwordsMismatch: true };
};
form = this.fb.group({ password: [''], confirm: [''] }, { validators: passwordsMatch });

// async (e.g. check username availability)
export function usernameAvailable(api: UserApi): AsyncValidatorFn {
  return control => timer(400).pipe(                     // debounce
    switchMap(() => api.isTaken(control.value)),
    map(taken => (taken ? { usernameTaken: true } : null)),
    catchError(() => of(null)),
  );
}
username = this.fb.control('', {
  validators: [Validators.required],
  asyncValidators: [usernameAvailable(inject(UserApi))],
  updateOn: 'blur',                                      // 'change' (default) | 'blur' | 'submit'
});
```
While async validators run, `control.pending` is `true`.

## 5. Custom form controls — `ControlValueAccessor`

Make your own component work with `formControlName` / `ngModel` (e.g. star rating, date picker).

```ts
import { Component, forwardRef, signal } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-star-rating',
  template: `
    @for (star of [1, 2, 3, 4, 5]; track star) {
      <button type="button" [disabled]="disabled()" (click)="select(star)" (blur)="onTouched()">
        {{ star <= value() ? '★' : '☆' }}
      </button>
    }
  `,
  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => StarRating), multi: true }],
})
export class StarRating implements ControlValueAccessor {
  value = signal(0);
  disabled = signal(false);
  private onChange: (v: number) => void = () => {};
  protected onTouched: () => void = () => {};

  select(v: number) { this.value.set(v); this.onChange(v); this.onTouched(); }

  // Forms API → component
  writeValue(v: number | null) { this.value.set(v ?? 0); }
  registerOnChange(fn: (v: number) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
  setDisabledState(d: boolean) { this.disabled.set(d); }
}
```
```html
<app-star-rating formControlName="rating" />
```
React equivalent: a controlled component with `value` + `onChange` props, registered with RHF's
`Controller`.

## 6. Signal Forms (stable in v22)

Signal Forms combine the typing of Reactive Forms, the simplicity of template-driven forms, and
signals. **Your data model is a plain signal**; `form()` wraps it with field state (value, valid,
touched, dirty, errors) and a **schema** of validation rules. Edits flow straight back into the
model signal.

Introduced as experimental in v21, production-ready in v22.

### Basic form
```ts
import { Component, signal } from '@angular/core';
import { form, FormField, required, email, minLength } from '@angular/forms/signals';

@Component({
  selector: 'app-login',
  imports: [FormField],
  template: `
    <form (submit)="submit($event)">
      <input type="email" [formField]="loginForm.email" />
      @if (loginForm.email().invalid() && loginForm.email().touched()) {
        @for (err of loginForm.email().errors(); track err.kind) {
          <small>{{ err.message }}</small>
        }
      }

      <input type="password" [formField]="loginForm.password" />

      <button type="submit" [disabled]="loginForm().invalid()">Log in</button>
    </form>
  `,
})
export class Login {
  model = signal({ email: '', password: '' });            // ← the single source of truth

  loginForm = form(this.model, path => {                  // ← schema: validation rules per path
    required(path.email, { message: 'Email is required' });
    email(path.email, { message: 'Invalid email' });
    minLength(path.password, 8, { message: 'At least 8 characters' });
  });

  submit(e: Event) {
    e.preventDefault();
    if (this.loginForm().invalid()) return;
    console.log(this.model());                            // already up to date, no getRawValue()
  }
}
```

### Reading state
```ts
loginForm()                 // root field state: .valid() .invalid() .dirty() .touched() .pending()
loginForm.email()           // field state for one path
loginForm.email().value()   // a WritableSignal for that field
loginForm.email().errors()  // [{ kind: 'required', message: '...' }, ...]
this.model()                // the whole value object
```
Nested objects and arrays work the same way: `form.address.city`, `form.items[0].qty`.

### More schema rules
```ts
form(this.model, path => {
  required(path.name);
  maxLength(path.name, 50);
  min(path.age, 18);
  pattern(path.zip, /^\d{5}$/);

  // conditional rules
  required(path.company, { when: ({ valueOf }) => valueOf(path.isBusiness) });
  disabled(path.discountCode, ({ valueOf }) => !valueOf(path.hasCoupon));

  // custom validator: return an error object or null/undefined
  validate(path.username, ({ value }) =>
    value().includes(' ') ? { kind: 'noSpaces', message: 'No spaces allowed' } : null,
  );
});
```
There are also async validation helpers (`validateAsync`, `validateHttp`) for server checks, and
custom controls implement the `FormValueControl` interface (typically a `value = model()` input),
which is simpler than `ControlValueAccessor`.

> Exact helper names and option shapes have evolved from v21 to v22 (e.g. the binding directive
> went from `[field]`/`Field` to `[formField]`/`FormField`). Use the angular.dev Signal Forms guide
> for your version as the source of truth.

### Signal Forms vs Reactive Forms

| | Reactive Forms | Signal Forms |
|---|---|---|
| Model | Lives inside `FormGroup` | Your own signal |
| Read value | `form.value` / `getRawValue()` | `model()` |
| React to changes | `valueChanges` Observable | `computed()` / `effect()` |
| Bind | `[formGroup]` + `formControlName` | `[formField]="f.path"` |
| Validators | `Validators.required` arrays | `required(path.x)` in a schema function |
| Custom control | `ControlValueAccessor` | `FormValueControl` |

## 7. Gotchas

1. **Forgot to import `ReactiveFormsModule`/`FormsModule`** → "Can't bind to 'formGroup' since it
   isn't a known property".
2. **`name` attribute is required** with `ngModel` inside a `<form>`.
3. **Don't mix** `ngModel` and `formControlName` on the same element (deprecated/error-prone).
4. `form.value` **excludes disabled controls**. Use `getRawValue()`.
5. Use `NonNullableFormBuilder` (or `nonNullable: true`), otherwise `reset()` sets values to `null`
   and types become `string | null`.
6. `valueChanges` emits **before** the parent's value updates if you subscribe on a child control;
   read the emitted value rather than `form.value`.

## 8. Interview questions

1. **Template-driven vs reactive vs signal forms?** Template-driven: the model lives in the
   template, it's async, simple, and less testable. Reactive: an explicit model in the class,
   synchronous access, typed, easy to test, dynamic, built on observables. Signal Forms (stable in
   v22): the model is a signal you own, validation is a schema function, binding uses
   `[formField]`, and it's the recommended choice for new forms.
2. **`FormControl` vs `FormGroup` vs `FormArray`?** A single value; a fixed set of named controls;
   a dynamic list of controls.
3. **How do you write a custom validator? Async validator?** A function returning `ValidationErrors | null`
   (or an Observable/Promise of it for async).
4. **What is `ControlValueAccessor`?** The interface that bridges a custom component and the Forms
   API: `writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`.
5. **`setValue` vs `patchValue`?** `setValue` requires every field; `patchValue` accepts a subset.
6. **What are typed forms?** Since v14, controls infer their value types; `NonNullableFormBuilder`
   avoids `| null`.
7. **How do you validate across fields?** A validator on the parent `FormGroup`.
8. **`updateOn: 'blur'`?** Value and validation update on blur instead of every keystroke.
