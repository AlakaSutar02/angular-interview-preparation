### **Q1. "Template-driven vs. Reactive Forms — when do you pick which?"**

> **How to answer:**
> "I default to **Reactive Forms** for almost everything in enterprise applications. Reactive forms are immutable, predictable, and fully testable without requiring a DOM render. They give us strict TypeScript type safety (`NonNullableFormBuilder`) and allow complex synchronous and asynchronous validations to be written as pure functions.
> I only pick **Template-driven forms** for simple, isolated UI controls—like a standalone search filter or a simple login box—where building an explicit `FormGroup` in TypeScript feels like unnecessary boilerplate."

**Example:**

```typescript
// Strongly-typed Reactive Form (Preferred)
const loginForm = fb.nonNullable.group({
  email: ["", [Validators.required, Validators.email]],
  password: ["", [Validators.required]],
});
```

---

### **Q2. "What's the difference between `form.value` and `form.getRawValue()`?"**

> **How to answer:**
> "`form.value` excludes disabled controls from the output object, whereas `form.getRawValue()` returns the full object payload including disabled fields.
> This is critical when sending data to a backend API. For instance, if a `userId` or `readOnlyTaxRate` field is disabled in the UI so the user can't edit it, `form.value` drops that field entirely. You almost always want `form.getRawValue()` when building submission payloads."

**Example:**

```typescript
const form = new FormGroup({
  id: new FormControl({ value: 101, disabled: true }),
  name: new FormControl("Alice"),
});

console.log(form.value); // { name: 'Alice' }
console.log(form.getRawValue()); // { id: 101, name: 'Alice' }
```

---

### **Q3. "How do you write an async validator?"**

> **How to answer:**
> "An async validator is a function returning an `Observable` or `Promise` of `ValidationErrors | null`. You pass it in the third parameter slot of a `FormControl`.
> Two crucial rules in production:
>
> 1. **Debounce the network calls** (e.g., `debounceTime(300)`) so you don't hit the server on every keystroke.
> 2. **Catch network errors gracefully** and return `null` (or a fallback error key). If an HTTP error throws uncaught, the control gets stuck in the `'PENDING'` status forever."

**Example:**

```typescript
export function checkUsernameUnique(api: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value) return of(null);
    return timer(300).pipe(
      switchMap(() => api.checkUsername(control.value)),
      map((isTaken) => (isTaken ? { usernameExists: true } : null)),
      catchError(() => of(null)), // Prevent control from staying PENDING on 500 error
    );
  };
}
```

---

### **Q4. "How do you validate relationships between two fields (e.g., Password vs. Confirm Password)?"**

> **How to answer:**
> "You attach a **cross-field validator to the parent `FormGroup**`, not to the individual `FormControl`s.
The validator receives the parent `FormGroup`, reads both sibling controls, compares their values, and returns an error object on the group's `.errors` property if they don't match."

**Example:**

```typescript
export const matchPasswordsValidator: ValidatorFn = (
  group: AbstractControl,
): ValidationErrors | null => {
  const password = group.get("password")?.value;
  const confirm = group.get("confirmPassword")?.value;
  return password === confirm ? null : { passwordMismatch: true };
};

// Attached at the FormGroup level
const form = fb.group(
  {
    password: [""],
    confirmPassword: [""],
  },
  { validators: matchPasswordsValidator },
);
```

---

### **Q5 (Trap). "What's the difference between `dirty` and `touched`?"**

> **How to answer:**
> "\* `dirty` means the user actively **changed the value** inside the field.
>
> - `touched` means the field **lost focus** (`blur` event).
>
> They are completely independent state flags. A common UX bug is showing red validation messages the second a page loads because a developer checked `control.invalid` or `control.dirty` instead of `control.touched`. The golden rule for displaying error messages on blur is: `control.invalid && (control.touched || formSubmitted)`."

---

### **Q6. "Why did my `valueChanges` subscription create an infinite loop?"**

> **How to answer:**
> "Because inside the `valueChanges` callback, you called `patchValue()` or `setValue()`, which emitted _another_ `valueChanges` event, creating an infinite loop.
> You fix it by passing `{ emitEvent: false }` inside your `patchValue()` call so it updates the internal control value silently without triggering downstream subscribers."

**Example:**

```typescript
this.form.get("category")?.valueChanges.subscribe((cat) => {
  // Pass emitEvent: false to prevent infinite loops!
  this.form.get("subcategory")?.patchValue("", { emitEvent: false });
});
```

---

### **Q7. "How does a custom form control participate in `formControlName`?"**

> **How to answer:**
> "By implementing the **`ControlValueAccessor` (CVA)** interface. You implement four mandatory methods:
>
> 1. `writeValue(obj)` — Angular passes values from the form model into your custom component.
> 2. `registerOnChange(fn)` — Registers a callback to notify Angular when the user changes the UI value.
> 3. `registerOnTouched(fn)` — Registers a callback to notify Angular when the input loses focus.
> 4. `setDisabledState(isDisabled)` — Handles enabling/disabling the inner UI.
>
> Finally, you register the component class in `providers: []` using `NG_VALUE_ACCESSOR` with `{ multi: true }` so Angular's form module discovers it."

---

### **Q8. "How do you dynamically add rows to a form?"**

> **How to answer:**
> "Using a **`FormArray`**. You instantiate a `FormArray` inside your group, push or remove `FormGroup` instances on array events (`array.push()`, `array.removeAt(i)`), and iterate over `array.controls` in the template using `@for`."

**Example:**

```typescript
// Component TS
items = fb.array([ fb.group({ name: [''], price: [0] }) ]);

addItem() {
  this.items.push(fb.group({ name: [''], price: [0] }));
}

```

```html
<!-- Template -->
@for (item of items.controls; track $index; let i = $index) {
<div [formGroupName]="i">
  <input formControlName="name" />
  <input formControlName="price" />
</div>
}
```

---

### **Q9. "How do you disable a control based on another control's value?"**

> **How to answer:**
> "In traditional reactive forms, you subscribe to the source control's `valueChanges` and call `.enable()` or `.disable()`—passing `{ emitEvent: false }`.
> In modern Angular, you can wrap `valueChanges` into a signal using `toSignal()` and run an `effect()` or derive disabled state declaratively."

**Example:**

```typescript
// Modern Signal Approach
const isCompany = toSignal(this.form.controls.type.valueChanges);

effect(() => {
  if (isCompany() === "BUSINESS") {
    this.form.controls.vatNumber.enable({ emitEvent: false });
  } else {
    this.form.controls.vatNumber.disable({ emitEvent: false });
  }
});
```

---

### **Q10. "How do you unit test a reactive form?"**

> **How to answer:**
> "You instantiate the component class directly via `TestBed`, grab the form instance off the component, call `.patchValue()`, and assert directly on `form.valid`, `form.errors`, or individual control values.
> You **do not need DOM interactions** (like typing into `<input>` tags or querying `DebugElement`) to test form validation rules—that's the core architectural advantage of reactive forms."

---

### **Q11. "What's `updateOn: 'submit'` for?"**

> **How to answer:**
> "It defers validation and `valueChanges` emissions until the user fires the submit event (or when a field loses focus if set to `updateOn: 'blur'`).
> It's useful for large enterprise forms or heavy forms with expensive async validations where validating every single keystroke creates UI lag or premature red error messages."

**Example:**

```typescript
const form = new FormGroup({
  bio: new FormControl("", { updateOn: "blur" }), // Validated on blur
  terms: new FormControl(false, { updateOn: "submit" }), // Validated on submit
});
```

---

### **Q12 (Advanced). "Signal-based forms — what's coming?"**

> **How to answer:**
> "Angular is exploring a **Signal-first Forms API** where form state, values, and validation statuses are primitive signals rather than RxJS Observables.
> Instead of subscribing to `valueChanges`, you'll read form values directly as reactive signals (`form.value()`, `form.status()`), eliminating subscription management completely while remaining fully interoperable with current reactive form abstractions."

---
