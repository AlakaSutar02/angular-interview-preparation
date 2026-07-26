Q. How is your application managing state and sharing data between components?

"In our application, we follow a Hybrid State Management Strategy designed for low complexity, high maintainability, and optimal runtime performance."

A. Global & Feature State (@ngrx/signals / SignalStore)
_"For application-wide domain state and complex feature data, we use @ngrx/signals (SignalStore). It gives us a strongly-typed, reactive, and boilerplate-free state container.
B. Shared Data Between Components (Smart / Dumb Architecture)
_"For component-to-component communication, we strictly enforce the Smart (Container) vs. Dumb (Presentational) pattern:

Parent-Child Communication: Data flows down via Signal Inputs (input.required()) and events flow up via Signal Outputs (output()).

Cross-Component / Sibling Communication: Siblings never communicate directly. Instead, they interact via a shared Service or the feature's SignalStore. Component A dispatches an action/method to the store, and Component B automatically updates because it reads a computed() signal derived from that store."\*

Q. How are you securing your Angular application?
"When it comes to securing our Angular apps, I focus on a few practical layers that actually matter in production:

    1. Handling Tokens & CSRF Realistically
    First, where we store tokens matters. Storing sensitive JWTs in localStorage opens you up to XSS attacks if any third-party script gets compromised. In our apps, we keep short-lived access tokens in memory (inside a service or state store) and use HttpOnly, SameSite=Strict cookies for refresh tokens so JavaScript can't touch them. We also configure Angular's HttpClient to automatically pass CSRF tokens (X-XSRF-TOKEN) with every API call.

    2. Stopping XSS at the Root
    Angular does a great job out of the box by escaping values in templates ({{ }}), but team members can accidentally introduce risks.

    We enforce a strict rule: No direct DOM manipulation using ElementRef.nativeElement.

    If we must render raw HTML from a CMS or rich-text editor using [innerHTML], we run it through DOMPurify to sanitize it first, rather than just blindly bypassing Angular’s sanitizer with bypassSecurityTrustHtml.

    3. Enforcing Access Control at the Route Level
    We use functional Route Guards (CanActivateFn) to check roles before a user lands on a page. More importantly, because we lazy-load our feature modules, these guards prevent unauthorized users from even downloading the JavaScript code for admin or internal pages they shouldn't see.

    4. Protecting the Environment (CSP & CI/CD)
    On the server/Nginx side, we set a strong Content Security Policy (CSP) header to restrict where scripts can be executed from. And in our CI/CD pipeline, we run tools like npm audit or Snyk to catch vulnerable npm packages before code gets merged to production."

Q. Where do you store authentication tokens? sub question Session Storage vs Local Storage and Can tokens be stored in a Service using RxJS Observables?

For token storage, we aim for HttpOnly cookies for refresh tokens and keep short-lived access tokens in-memory inside an AuthService using RxJS BehaviorSubject or Signals. Between local storage and session storage, sessionStorage is safer for tokens because it automatically clears when the tab closes and stays isolated per tab.
Here is the concise, ultra-focused version:

---

### **2. Session Storage vs. Local Storage**

| Feature              | `sessionStorage`                                                                   | `localStorage`                                                            |
| -------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **Lifespan**         | Clears automatically when the **tab closes**.                                      | Persists until explicitly deleted.                                        |
| **Tab Isolation**    | **Isolated per tab** (Opening a new tab = fresh state).                            | **Shared across all tabs** under the same domain.                         |
| **Security Verdict** | **Safer fallback for tokens.** Limits exposure time and isolates sessions per tab. | Risky. Retains tokens indefinitely and exposes them across all open tabs. |

---

### **3. Can tokens be stored in a Service using RxJS Observables?**

**Yes.** This is the exact implementation used for **in-memory storage**.

```typescript
@Injectable({ providedIn: "root" })
export class AuthService {
  // In-Memory Token Stream
  private accessToken$ = new BehaviorSubject<string | null>(null);

  setToken(token: string): void {
    this.accessToken$.next(token);
  }

  getToken(): string | null {
    return this.accessToken$.getValue(); // Fast synchronous access for HTTP Interceptors
  }
}
```

- **How it works:** On login, push the token into the `BehaviorSubject`. The `HttpInterceptor` reads `authService.getToken()` synchronously and attaches it to `Authorization: Bearer <token>` headers without ever exposing it to `localStorage`.

Q. Write a function to fetch data from an API (JsonPlaceholder API was used). How do you design API calling from the UI? When do you use the Async Pipe? When should you unsubscribe manually?

---

### Part 1: Writing the API Call (JSONPlaceholder Example)

In modern Angular (v16+), we use standalone functions with `HttpClient` and convert API streams to Signals or use `toSignal` for clean template consumption.

#### **Service Implementation (`user.service.ts`)**

```typescript
import { Injectable, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";

export interface User {
  id: number;
  name: string;
  email: string;
  company: { name: string };
}

@Injectable({ providedIn: "root" })
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = "https://jsonplaceholder.typicode.com/users";

  // Returns an Observable stream
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }
}
```

#### **Component Usage (`user-list.component.ts`)**

```typescript
import { Component, inject } from "@angular/core";
import { AsyncPipe } from "@angular/common";
import { UserService } from "./user.service";

@Component({
  selector: "app-user-list",
  standalone: true,
  imports: [AsyncPipe],
  template: `
    <ul>
      @for (user of users$ | async; track user.id) {
        <li>{{ user.name }} ({{ user.email }})</li>
      } @empty {
        <li>No users found.</li>
      }
    </ul>
  `,
})
export class UserListComponent {
  private userService = inject(UserService);

  // Expose stream directly to template
  readonly users$ = this.userService.getUsers();
}
```

---

### Part 2: Designing API Calls, Async Pipe, and Manual Unsubscription

#### **1. How do you design API calling from the UI?**

We follow a **3-Layer Architecture**:

```
UI Component (Presenter) ──> Service / Facade (Orchestrator) ──> HttpClient (Data Layer)

```

1. **Separation of Concerns:** Components **never** call `HttpClient` directly or construct endpoints. Components trigger actions or subscribe to state.
2. **Immutability & Reactive Streams:** Services handle data fetching, mapping (`map`), error catches (`catchError`), and caching (`shareReplay(1)`).
3. **Declarative over Imperative:** Avoid calling `.subscribe()` inside component TypeScript code to assign local variables. Instead, bind Observables directly to templates via `AsyncPipe` or convert them using `toSignal()`.

---

#### **2. When do you use the `AsyncPipe`?**

Use the `AsyncPipe` (or modern `toSignal`) whenever you need to render asynchronous Observable streams directly in the template.

- **Automatic Lifecycle Management:** It automatically subscribes when the component initializes and **unsubscribes when the component is destroyed**.
- **OnPush Compatibility:** It marks the component for check (`markForCheck()`) automatically whenever the Observable emits a new value, making it ideal for `ChangeDetectionStrategy.OnPush`.
- **Zero Boilerplate:** Prevents writing manual class fields, `ngOnInit` subscriptions, and `ngOnDestroy` cleanup handlers.

---

#### **3. When should you unsubscribe manually?**

You must unsubscribe manually whenever a subscription **outlives the component lifecycle** and is **not managed by Angular's template engine**.

#### **Must Unsubscribe:**

- **Infinite / Long-Lived Observables:** Subscriptions to `router.events`, RxJS `interval()`, `fromEvent()` DOM listeners, or custom WebSocket streams/Subjects inside `.ts` files.
- **Why:** If not unsubscribed, the closure retains references to the component instance, preventing Garbage Collection and causing **memory leaks**.

#### **How to Unsubscribe (Modern Best Practices):**

- **Option A: `takeUntilDestroyed` (Angular 16+)**

```typescript
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

export class UserComponent {
  constructor(private userService: UserService) {
    this.userService
      .getUsers()
      .pipe(
        takeUntilDestroyed(), // Automatically cleans up when DestroyRef triggers
      )
      .subscribe((users) => console.log(users));
  }
}
```

- **Option B: RxJS `takeUntil` with Subject (Legacy)**

```typescript
export class UserComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.userService.getUsers().pipe(takeUntil(this.destroy$)).subscribe();
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

#### **DO NOT Need Manual Unsubscription:**

1. **HttpClient requests:** Angular's `HttpClient` automatically completes the Observable after emitting the single response/error payload.
2. **`AsyncPipe` / `toSignal()`:** Angular handles the teardown internally when the view is destroyed.

Q. Explain different types of Signals and the use case of computed().
Here is a clear breakdown of the **3 core types of Signals** in Angular and how/when to use `computed()`.

---

## 1. The 3 Types of Signals

Angular provides three primary signal primitives to manage reactivity:

```
                  ┌──────────────────────┐
                  │   Writable Signal    │  (Source of Truth / Mutatable)
                  └──────────┬───────────┘
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
┌──────────────────────┐          ┌──────────────────────┐
│   Computed Signal    │          │        Effect        │
│ (Derived State / Read)│          │ (Side Effect/Syncing)│
└──────────────────────┘          └──────────────────────┘

```

---

### A. Writable Signals (`signal()`)

- **What it is:** A mutable state container holding a primary value. It serves as your source of truth.
- **Mutations:** Modified using `.set()` (replaces value) or `.update()` (modifies based on previous value).
- **Example:**

```typescript
const price = signal<number>(100);
const quantity = signal<number>(1);

price.set(120); // Overwrites to 120
quantity.update((q) => q + 1); // Updates 1 -> 2
```

---

### B. Computed Signals (`computed()`)

- **What it is:** A **read-only** signal that derives its value from other signals.
- **Key Characteristic:** It automatically tracks its dependencies and recalculates only when those dependencies change.
- **Example:**

```typescript
// Derived automatically from price and quantity signals
const totalPrice = computed(() => price() * quantity());
```

---

### C. Effects (`effect()`)

- **What it is:** A reactive operation that runs whenever any signal read inside its context changes.
- **Key Characteristic:** Used **strictly for side effects** (logging, syncing with `localStorage`, third-party library calls, analytics), **never** for state mutations.
- **Example:**

```typescript
effect(() => {
  console.log(`Cart total updated to: $${totalPrice()}`);
  localStorage.setItem("cart_total", totalPrice().toString());
});
```

---

## 2. Use Case of `computed()`

### **Primary Purpose**

`computed()` is used to create **derived, read-only reactive state** without manually managing subscriptions, listeners, or state synchronization.

---

### **Why Use `computed()`? (Key Benefits)**

1. **Lazy Evaluation:**

- It calculates its value **only when read** (e.g., when rendered in the DOM or called in code). If the underlying signals change 10 times but the UI isn't rendering it, the function doesn't execute 10 times.

2. **Automatic Dependency Tracking:**

- You don't specify dependency arrays (like React's `useMemo`). Angular dynamically tracks whichever signals you invoke inside the callback.

3. **Glitch-Free Memoization & Caching:**

- The result is cached. Calling `totalPrice()` 5 times in a template executes the math **once** and returns the cached result for the remaining 4 calls until a dependency mutates.

---

### **Real-World Practical Example: Filtered Data List**

Without `computed()`, filtering a list requires listening to inputs and search queries manually. With `computed()`, it is declarative and automatic:

```typescript
import { Component, signal, computed } from "@angular/core";

interface Product {
  id: number;
  name: string;
  category: string;
}

@Component({
  selector: "app-product-list",
  standalone: true,
  template: `
    <input
      type="text"
      [value]="searchTerm()"
      (input)="updateSearch($event)"
      placeholder="Search..."
    />

    <p>
      Showing {{ filteredProducts().length }} of
      {{ products().length }} products
    </p>

    <ul>
      @for (product of filteredProducts(); track product.id) {
        <li>{{ product.name }}</li>
      }
    </ul>
  `,
})
export class ProductListComponent {
  // Writable Signals (Raw State)
  readonly products = signal<Product[]>([
    { id: 1, name: "Laptop", category: "Electronics" },
    { id: 2, name: "Headphones", category: "Electronics" },
    { id: 3, name: "Desk Chair", category: "Furniture" },
  ]);

  readonly searchTerm = signal<string>("");

  // Derived Signal (Computed State)
  readonly filteredProducts = computed(() => {
    const term = this.searchTerm().toLowerCase();
    return this.products().filter((p) => p.name.toLowerCase().includes(term));
  });

  updateSearch(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.searchTerm.set(input.value);
  }
}
```

---

> _"Angular provides three Signal types: **Writable Signals** for managing raw mutable state, **Computed Signals** for derived state, and **Effects** for side-effects. We use `computed()` whenever we need to transform or filter existing state. It provides **automatic dependency tracking, lazy evaluation, and built-in memoization**."_

Q. How would you track every route change to monitor user navigation?

Here is the complete **Modern Angular (v16–v19+) code implementation** for a Navigation Tracker service. It uses **Standalone Services**, **Functional `inject()**`, **Signals**, and **`takeUntilDestroyed()`\*\*.

---

### **`navigation-tracker.service.ts`**

```typescript
import { Injectable, inject, signal } from "@angular/core";
import {
  Router,
  NavigationEnd,
  NavigationStart,
  NavigationError,
} from "@angular/router";
import { filter } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

@Injectable({ providedIn: "root" })
export class NavigationTrackerService {
  private router = inject(Router);

  // Signal to expose the current active URL reactively across the app
  readonly currentUrl = signal<string>("");

  // Signal to track loading state during route transitions
  readonly isNavigating = signal<boolean>(false);

  constructor() {
    // 1. Track Successful Route Completion
    this.router.events
      .pipe(
        filter(
          (event): event is NavigationEnd => event instanceof NavigationEnd,
        ),
        takeUntilDestroyed(), // Modern RxJS Interop: Automatically unbinds on app destruction
      )
      .subscribe((event) => {
        this.isNavigating.set(false);
        this.currentUrl.set(event.urlAfterRedirects);

        // Send to Analytics / Monitoring Service
        this.logPageView(event.urlAfterRedirects);
      });

    // 2. Optional: Track Start & Errors for Performance & Monitoring
    this.router.events
      .pipe(
        filter(
          (event) =>
            event instanceof NavigationStart ||
            event instanceof NavigationError,
        ),
        takeUntilDestroyed(),
      )
      .subscribe((event) => {
        if (event instanceof NavigationStart) {
          this.isNavigating.set(true);
        } else if (event instanceof NavigationError) {
          this.isNavigating.set(false);
          console.error("Route Navigation Failed:", event.error);
        }
      });
  }

  private logPageView(url: string): void {
    // Analytics Payload Execution (e.g., Google Analytics, Application Insights)
    console.log(`[Analytics] Page View Tracked: ${url}`);
  }
}
```

---

### **Initializing in `app.config.ts**`

To ensure tracking begins immediately when the application bootstraps:

```typescript
import {
  ApplicationConfig,
  provideAppInitializer,
  inject,
} from "@angular/core";
import { provideRouter, withComponentInputBinding } from "@angular/router";
import { routes } from "./app.routes";
import { NavigationTrackerService } from "./navigation-tracker.service";

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),

    // Modern Angular (v18+) Provider to initialize tracking on app boot
    provideAppInitializer(() => {
      inject(NavigationTrackerService);
    }),
  ],
};
```

---

1. **`takeUntilDestroyed()`**: Replaces legacy `Subject.next()` / `ngOnDestroy` boilerplates for automatic subscription cleanup.
2. **`signal()` State**: Exposes `currentUrl` and `isNavigating` as Signals for high-performance, fine-grained UI binding (e.g., showing a global progress bar).
3. **`inject()`**: Uses modern dependency injection rather than constructor parameter lists.
4. **`provideAppInitializer()`**: Replaces the deprecated `APP_INITIALIZER` injection token pattern.

Q. Suppose a Reactive Form contains a dropdown that gets enabled/disabled from multiple events. A defect is reported where the dropdown behaves incorrectly. How would you debug which event is changing its state?
Here is a direct, polished answer you can speak word-for-word in an interview:

---

> "When a Reactive Form control is being enabled or disabled from multiple events, `statusChanges` alone won't reveal _who_ made the call. I debug and fix this using three steps:
> **1. Print Execution Call Stacks**
> In `ngOnInit`, I subscribe to `control.statusChanges` and log `console.trace()`. Whenever the status flips between `VALID` and `DISABLED`, the console prints the exact stack trace leading to that mutation.
> **2. Monkey-Patch `.enable()` and `.disable()` for Breakpoints**
> If I need to inspect runtime variables right at the moment of change, I temporarily override the control instance's `enable` and `disable` methods in `ngOnInit`:
>
> ```typescript
> const control = this.form.controls.myDropdown;
> const originalDisable = control.disable.bind(control);
> control.disable = (...args) => {
>   debugger; // Pauses execution instantly in DevTools
>   return originalDisable(...args);
> };
> ```
>
> Pausing execution directly on `debugger` lets me look at the **Call Stack panel** in Chrome DevTools to see the exact event listener, API callback, or stream subscription that invoked it.
> **3. Refactor to a Single Source of Truth**
> Once found, to permanently fix the root cause, I remove the scattered imperative `.enable()` and `.disable()` calls. Instead, I consolidate the logic into a single `computed()` signal or RxJS stream, using an Angular `effect()` to sync the control's enabled/disabled state in one centralized place."

Q. Your UI development is complete, but backend APIs are still under development. API contracts are available. How will you test your application?
(Expected: Mocking APIs)

When backend APIs aren't ready but API contracts are defined, we unblock UI development and testing by mocking the API layer based on those schemas\

Q. Your form has more than 50 fields. Will you keep the complete Reactive Form and template inside one component?
"No, absolutely not. Putting a 50+ field form inside a single component creates severe maintainability issues, huge template files, and potential performance bottlenecks during change detection.

Instead, I break it down using a Modular Form Architecture:

1. Logical Feature/Step Decomposition
   First, I group the 50+ fields into smaller, domain-driven logical sections (e.g., PersonalInfoStep, AddressStep, PaymentDetailsStep).

2. Smart Parent / Dumb Child Pattern with ControlContainer

Parent Component (Smart): Creates and owns the root FormGroup (or FormArray) and orchestrates the submission pipeline.

Child Components (Dumb): Each child represents a section of the form. Instead of passing form controls down through @Input(), child components inject the parent's FormGroup directly using Angular's ControlContainer or FormGroupDirective. This allows child components to bind directly to their sub-FormGroup without coupling logic.

3. Stepper / Tabbed UI UX
   A 50-field single page is poor UX. I structure it as a multi-step wizard or tabbed layout, rendering sections on demand using @if or lazy-loaded routes.

4. Performance Tuning
   For large forms, listening to global form.valueChanges can degrade performance. I isolate value changes to specific sub-controls and use ChangeDetectionStrategy.OnPush across all child form components."

Q. How would you build a dynamic form using JSON configuration instead of hardcoding controls?

### **Interview Answer (Word-for-Word Script)**

> "To build a dynamic form driven by JSON configuration, I design a **Schema-Driven Dynamic Form Engine**. Instead of hardcoding `FormControl` instances in HTML, we map JSON objects into form controls dynamically and render them using custom field components.
> The architecture relies on three core steps:
> **1. Define a Strongly-Typed JSON Schema Model**
> We define a TypeScript interface for the JSON fields (specifying key, type, label, validations, options, and conditional visibility rules).
> **2. Build a Dynamic Form Service Factory**
> A dedicated service iterates over the JSON schema array and dynamically instantiates a `FormGroup` containing `FormControl`s with their corresponding Angular `Validators` (e.g., `required`, `email`, `pattern`).
> **3. Render Using a Dynamic Host Component (`@switch` or `NgComponentOutlet`)**
> A parent renderer loops through the schema using `@for`. Inside, it renders inputs dynamically—either using a native `@switch` on field types (`text`, `select`, `checkbox`) or via `ngComponentOutlet` if fields require custom complex widgets.
> **4. Handle Conditional Logic & Dependencies**
> For conditional fields (e.g., _'Show Field B only if Field A == Yes'_), we listen to `form.valueChanges` or use Angular **Signals / `computed()**`to dynamically calculate`hidden`or`disabled` states without polluting component templates."

---

### **Complete Dynamic Form Engine Example**

#### **1. The JSON Schema Contract (`dynamic-form.model.ts`)**

```typescript
export interface FormFieldConfig {
  key: string;
  label: string;
  type: "text" | "number" | "select" | "checkbox";
  value?: any;
  options?: { label: string; value: any }[]; // For dropdowns
  validators?: {
    required?: boolean;
    min?: number;
    email?: boolean;
  };
}
```

#### **2. Form Generator Service (`dynamic-form.service.ts`)**

```typescript
import { Injectable } from "@angular/core";
import {
  FormGroup,
  FormControl,
  Validators,
  ValidatorFn,
} from "@angular/forms";
import { FormFieldConfig } from "./dynamic-form.model";

@Injectable({ providedIn: "root" })
export class DynamicFormService {
  buildFormGroup(config: FormFieldConfig[]): FormGroup {
    const group: Record<string, FormControl> = {};

    config.forEach((field) => {
      const validatorList: ValidatorFn[] = [];

      if (field.validators?.required) validatorList.push(Validators.required);
      if (field.validators?.email) validatorList.push(Validators.email);
      if (field.validators?.min !== undefined)
        validatorList.push(Validators.min(field.validators.min));

      group[field.key] = new FormControl(field.value ?? "", validatorList);
    });

    return new FormGroup(group);
  }
}
```

#### ** 3. Dynamic Form Renderer Component (`dynamic-form.component.ts`) **

```typescript
import { Component, Input, OnInit, inject } from "@angular/core";
import { FormGroup, ReactiveFormsModule } from "@angular/forms";
import { FormFieldConfig } from "./dynamic-form.model";
import { DynamicFormService } from "./dynamic-form.service";

@Component({
  selector: "app-dynamic-form",
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    @if (form) {
      <form [formGroup]="form" (ngSubmit)="onSubmit()">
        @for (field of config; track field.key) {
          <div class="form-field">
            <label [for]="field.key">{{ field.label }}</label>

            @switch (field.type) {
              @case ("text") {
                <input
                  [id]="field.key"
                  type="text"
                  [formControlName]="field.key"
                />
              }
              @case ("select") {
                <select [id]="field.key" [formControlName]="field.key">
                  @for (opt of field.options; track opt.value) {
                    <option [value]="opt.value">{{ opt.label }}</option>
                  }
                </select>
              }
              @case ("checkbox") {
                <input
                  [id]="field.key"
                  type="checkbox"
                  [formControlName]="field.key"
                />
              }
            }

            <!-- Validation Error Handling -->
            @if (form.get(field.key)?.invalid && form.get(field.key)?.touched) {
              <span class="error">This field is invalid.</span>
            }
          </div>
        }

        <button type="submit" [disabled]="form.invalid">Submit</button>
      </form>
    }
  `,
})
export class DynamicFormComponent implements OnInit {
  private formService = inject(DynamicFormService);

  @Input({ required: true }) config: FormFieldConfig[] = [];
  form!: FormGroup;

  ngOnInit(): void {
    this.form = this.formService.buildFormGroup(this.config);
  }

  onSubmit(): void {
    if (this.form.valid) {
      console.log("Form Submitted Payload:", this.form.value);
    }
  }
}
```

---

1. **Custom / Complex Controls:** Mention that for complex controls (e.g., custom rich text editors or date pickers), you map field types to dynamic component classes using `NgComponentOutlet` rather than a simple `@switch`.
2. **Performance with Large Schemas:** For large JSON configs (50+ fields), track form controls using track keys (`track field.key`) to avoid full re-rendering of the DOM tree when field values mutate.
3. **Existing Open-Source Libraries:** Mention that in enterprise projects, you evaluate existing battle-tested libraries like **ngx-formly** before building a custom in-house engine.

Q. Your application has a Stepper, and each step is implemented in a different Angular component. How would you share the complete form across all steps?

"To share a form across multi-step components, I choose between two approaches depending on routing:

1. Standard Stepper (Nested Components): Parent Form + ControlContainer
   The parent stepper component owns the root FormGroup. Child step components inject the parent's form context using ControlContainer (skipSelf: true). This lets each child bind directly to its sub-FormGroup (e.g., formGroupName="address") with zero @Input() boilerplate or tight coupling.

2. Routed Stepper (Separate URLs): Shared FormStateService
   If steps live on distinct routes (/step-1, /step-2), ControlContainer can't bridge the components. Instead, we store the root FormGroup in a singleton FormStateService. Each routed component injects the service to read and update its specific form slice, preserving state across route navigations.

In both cases, step navigation is gated by validating only the active step's sub-FormGroup before advancing."
