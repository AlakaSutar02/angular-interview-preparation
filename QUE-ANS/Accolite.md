Here is the principal lead-level interview script, structured specifically to address each question with deep technical clarity, modern Angular 17â€“19 paradigms, and JavaScript internal mechanics.

---

### **1) Explain Angular Architecture (Focus on Folder Structure, Not Just Components/Modules)**

> **"When designing scalable enterprise Angular applications, I enforce a Domain-Driven Design (DDD) or Feature-Sliced folder architecture rather than grouping strictly by technical file types (e.g., placing all components in one folder and all services in another).**
>
> #### Recommended Folder Structure:\*\*
>
> ```text
> src/
> ├── app/
> │   ├── core/                  # Global singletons (interceptors, auth guards, app config)
> │   │   ├── guards/
> │   │   ├── interceptors/
> │   │   ├── services/
> │   │   └── core.providers.ts
> │   │
> │   ├── shared/                # Reusable UI primitives, generic pipes, custom directives
> │   │   ├── components/        # Input fields, buttons, modal dialogs
> │   │   ├── directives/
> │   │   └── pipes/
> │   │
> │   ├── features/              # Business Domain Modules (Lazy Loaded Boundaries)
> │   │   ├── dashboard/
> │   │   │   ├── components/    # Feature-specific sub-components
> │   │   │   ├── services/      # Feature state / API service
> │   │   │   ├── dashboard.routes.ts
> │   │   │   └── dashboard.component.ts
> │   │   └── user-profile/
> │   │
> │   ├── app.component.ts       # Root shell layout
> │   ├── app.config.ts          # Application-wide providers (Router, HttpClient, Hydration)
> │   └── app.routes.ts          # Top-level routing definitions
> │
> └── assets/                    # Static assets, SCSS mixins/tokens, i18n JSONs
>
> ```
>
> #### Key Architectural Rules:
>
> 1. **Domain Isolation:** Code inside `features/dashboard` must not directly import files from `features/user-profile`. Cross-feature communication happens via Shared Services or Router state.
> 2. **Shared vs Core:** `Core` holds singletons loaded once at app boot. `Shared` contains stateless or UI-focused artifacts imported across multiple features."

---

### **2) How Bootstrapping Works in Angular (`main.ts`) & What Changed with Standalone**

> **"Bootstrapping is the initialization process where Angular sets up its runtime environment, instantiates the root Injector, compiles the root view, and mounts it into the index.html DOM host tag.**
>
> #### Legacy Bootstrap (`NgModule` based):\*\*
>
> `main.ts` executed `platformBrowserDynamic().bootstrapModule(AppModule)`. `AppModule` processed its `@NgModule.bootstrap` array, registered global providers, and loaded the root `AppComponent`.
>
> #### Modern Standalone Bootstrap (Angular 17â€“19):\*\*
>
> Since there is no `AppModule`, bootstrapping starts directly with `bootstrapApplication()` passing a **Standalone Root Component** and a configuration object:
>
> ```typescript
> // main.ts
> import { bootstrapApplication } from "@angular/platform-browser";
> import { AppComponent } from "./app/app.component";
> import { appConfig } from "./app/app.config";
>
> bootstrapApplication(AppComponent, appConfig).catch((err) =>
>   console.error(err),
> );
>
> // app.config.ts
> import {
>   ApplicationConfig,
>   provideZonelessChangeDetection,
> } from "@angular/core";
> import { provideRouter } from "@angular/router";
> import { provideHttpClient, withInterceptors } from "@angular/common/http";
> import { routes } from "./app.routes";
> import { authInterceptor } from "./core/interceptors/auth.interceptor";
>
> export const appConfig: ApplicationConfig = {
>   providers: [
>     provideZonelessChangeDetection(), // Modern Zoneless setup
>     provideRouter(routes),
>     provideHttpClient(withInterceptors([authInterceptor])),
>   ],
> };
> ```
>
> **What Changed:** `ApplicationConfig` replaces module-level `providers`. Framework features (Router, HttpClient, SSR) are registered using explicit functional providers (`provideRouter()`, `provideHttpClient()`), making bundle initialization tree-shakable."

---

### **3) What is Dependency Injection (DI)?**

> **"Dependency Injection is a software design pattern where a class requests dependencies from an external framework provider rather than instantiating them internally using `new`.**
>
> #### Role in Angular:
>
> Angular provides a **Hierarchical DI Framework**:
>
> 1. **Inversion of Control (IoC):** Decouples classes from specific implementations, simplifying unit testing via mock injection.
> 2. **Token Resolution:** When a component requests a dependency (via constructor or `inject()`), Angular looks up the DI tree (ElementInjector $\rightarrow$ EnvironmentInjector $\rightarrow$ NullInjector) and supplies the cached singleton or instance.
>
> ````typescript
> // Modern Injection Syntax
> @Component({ ... })
> export class UserComponent {
>   private userService = inject(UserService); // Functional DI injection
> }
> ```"
>
> ````

---

### **4) Use Cases of Interceptor & API Mocking Awareness**

> **"An HTTP Interceptor inspects, transforms, or short-circuits outgoing requests and incoming responses globally.**
>
> #### Enterprise Use Cases:
>
> 1. **Authentication:** Attaching JWT `Bearer` tokens to outgoing API request headers.
> 2. **Global Error Handling:** Intercepting 401 (Unauthorized) or 403 (Forbidden) errors to handle token refreshes or route redirects.
> 3. **Request Profiling & Logging:** Calculating API execution time (`performance.now()`).
> 4. **Caching:** Returning cached response observables for duplicate GET requests.
>
> #### Mocking with Interceptors:\*\*
>
> Yes, interceptors are ideal for offline API mocking or local development testing before backend services are deployed.
>
> ```typescript
> import { HttpInterceptorFn, HttpResponse } from "@angular/common/http";
> import { of } from "rxjs";
> ```

> export const mockUserInterceptor: HttpInterceptorFn = (req, next) => {
> if (req.url.endsWith('/api/users') && req.method === 'GET') {
> const mockUsers = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }];
> return of(new HttpResponse({ status: 200, body: mockUsers })); // Intercepts and returns mock payload directly
> }
> return next(req); // Passes real API requests through
> };
>
> ```"
>
> ```

---

### **5) How Change Detection Works in Angular**

> **"Change Detection (CD) is the process by which Angular synchronizes internal component state with the rendered DOM template.**
>
> 1. **Zone.js Driven (Legacy/Default):** `Zone.js` patches async browser APIs (`setTimeout`, `fetch`, clicks). When an event finishes, Zone.js triggers global top-down change detection checking all components (`Default` strategy) or marked components (`OnPush` strategy).
> 2. **Signals Driven (Zoneless Modern):** Components read fine-grained `Signal` primitives. When a Signal value updates, it directly notifies the framework scheduler to mark **only the specific dependent component views as dirty**, bypassing top-down dirty checking."

---

### **6) What are Signals, Different Types of Signals, and How to Use `effect**`

> **"A Signal is a reactive value wrapper that notifies consumers synchronously when its value mutates.**

#### Types of Signals:

1. **Writable Signals (`signal(value)`):** State containers that can be mutated using `.set()` or `.update()`.
2. **Computed Signals (`computed(() => derivation)`):** Read-only, memoized derived reactive values.
3. **Linked Signals (`linkedSignal()`):** Writable signals whose value automatically resets when a source dependency mutates.
4. **Signal Inputs / Outputs (`input()`, `output()`):** Component reactive API bindings.

#### How to use `effect()`:

`effect()` schedules an operation that re-runs automatically whenever any read Signal within its scope changes. It is meant exclusively for **side effects** (DOM calls, logging, storage sync), not for updating other signals.

````typescript
import { Component, signal, effect } from '@angular/core';

@Component({ ... })
export class SettingsComponent {
  theme = signal<'dark' | 'light'>('light');

  constructor() {
    effect(() => {
      // Syncs theme state with localStorage whenever theme() updates
      localStorage.setItem('user-theme', this.theme());
      document.body.className = this.theme();
    });
  }

  toggleTheme() {
    this.theme.update(current => current === 'light' ? 'dark' : 'light');
  }
}
```



### Q. Which RxJS Operators Have You Used?

> "In production architectures, I categorize RxJS operators by their responsibilities:
>
> * **Higher-Order Mapping Operators:** `switchMap` (search/typeahead cancellation), `concatMap` (sequential queues), `mergeMap` (concurrent processing), `exhaustMap` (preventing double clicks).
> * **Combination Operators:** `forkJoin` (parallel page loads), `combineLatest` (reactive filter streams), `zip`.
> * **Filtering & Timing:** `debounceTime`, `distinctUntilChanged`, `filter`, `takeUntilDestroyed`.
> * **Utility & Error Handling:** `tap` (side-effects), `catchError` (error fallback), `retry` / `retryWhen` (resilience logic), `shareReplay` (caching)."

---

### Q. Difference Between `concatMap` and `mergeMap`

| Operator | Execution Order | Concurrency | Primary Production Use Case |
| :--- | :--- | :--- | :--- |
| **`concatMap`** | **Sequential.** Waits for the current inner Observable to complete before subscribing to the next. | $1$ active stream at a time. | FIFO queues (e.g., sequentially saving ordered form steps). |
| **`mergeMap`** | **Concurrent.** Subscribes to inner Observables immediately as they arrive. | Unlimited active streams running in parallel. | Parallel operations (e.g., fetching details for multiple items simultaneously). |

---

### Q. Difference Between Pure and Impure Pipes

> * **Pure Pipe (Default: `pure: true`):** Angular executes a pure pipe **ONLY when it detects a change in input value references** (primitive value changes or new object memory references). Highly performant.
> * **Impure Pipe (`pure: false`):** Angular executes an impure pipe on **EVERY single Change Detection run**, regardless of whether inputs changed.
> * **Warning:** Impure pipes that contain heavy operations can severely impact application rendering performance."

---

### Q. Which Will Execute First: `setTimeout` or `Promise`?

> **"`Promise` callback will execute first.**
>
> #### Reason (Event Loop Queue Priority):**
> * `Promise.then()` callbacks are placed in the **Microtask Queue**.
> * `setTimeout()` callbacks are placed in the **Macrotask (Callback) Queue**.
> * When the Call Stack clears, the Event Loop processes **all pending Microtasks** before picking the next Macrotask.

```javascript
setTimeout(() => console.log('Macrotask: setTimeout'), 0);
Promise.resolve().then(() => console.log('Microtask: Promise'));

// Output Execution Order:
// 1. "Microtask: Promise"
// 2. "Macrotask: setTimeout"
```"

---

### Q. What is the Event Loop?

> **"The Event Loop is JavaScript's single-threaded concurrency runtime model.**
>
> It continuously monitors the **Call Stack** and task queues. When the Call Stack is empty:
> 1. It drains the **Microtask Queue** completely (Promises, `queueMicrotask`, MutationObserver).
> 2. It processes one item from the **Macrotask Queue** (`setTimeout`, `setInterval`, DOM events).
> 3. It triggers UI rendering updates if needed, then repeats the loop."

---

### Q. What is a Closure?

> **"A Closure is the combination of a function bundled together with references to its surrounding lexical environment.**
>
> In JavaScript, closures allow an inner function to retain access to variables declared in its outer scope even after the outer function has returned and exited the Call Stack."

```javascript
function createCounter() {
  let count = 0; // Enclosed variable inside outer scope
  return function() {
    count++; // Accesses and preserves lexical 'count'
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2

````

---

### **13) What is Prototype and Which OOP Concept Does It Represent?**

> **"In JavaScript, every object has an internal link to another object called its `prototype`.** When attempting to access a property or method on an object, JS searches the object itself first; if missing, it traverses up the **Prototype Chain** until it finds the property or hits `null`.
> **OOP Concept Represented:** It represents **Inheritance** (specifically _Prototypal Inheritance_), enabling object instances to share methods and properties without duplicating them in memory."

---

### **14) Difference Between `let` and `var**`

| Feature                  | `var`                                     | `let`                                                                 |
| ------------------------ | ----------------------------------------- | --------------------------------------------------------------------- |
| **Scope**                | Function Scoped.                          | Block Scoped (`{}`).                                                  |
| **Hoisting**             | Hoisted and initialized with `undefined`. | Hoisted, but placed in a **Temporal Dead Zone (TDZ)** until declared. |
| **Re-declaration**       | Allowed within the same scope.            | Throws a SyntaxError.                                                 |
| **Global Window Object** | Attaches to `window.varName`.             | Does not attach to `window`.                                          |

---

### **15) Reverse a String Program (In-place / Algorithm Approach)**

```typescript
// Approach 1: Modern Functional Expression
function reverseStringBuiltIn(str: string): string {
  return str.split("").reverse().join("");
}

// Approach 2: Manual Two-Pointer Algorithm (O(N) Time, O(N) Space)
function reverseStringAlgorithm(str: string): string {
  const chars = str.split("");
  let left = 0;
  let right = chars.length - 1;

  while (left < right) {
    // Swap characters in place
    const temp = chars[left];
    chars[left] = chars[right];
    chars[right] = temp;
    left++;
    right--;
  }

  return chars.join("");
}

console.log(reverseStringAlgorithm("Angular")); // Output: "ralugnA"
```

---

### **16) Reusable Component Showcasing Input and Output (Modern Signal API)**

#### **Child Reusable Component (`custom-counter.component.ts`):**

```typescript
import { Component, input, output } from "@angular/core";

@Component({
  selector: "app-custom-counter",
  standalone: true,
  template: `
    <div class="counter-box">
      <h3>{{ label() }}</h3>
      <p>Current Count: {{ count() }}</p>
      <button (click)="increment()">Increment</button>
    </div>
  `,
  styles: [
    `
      .counter-box {
        border: 1px solid #ccc;
        padding: 1rem;
        borderradius: 8px;
      }
    `,
  ],
})
export class CustomCounterComponent {
  // Signal Inputs
  label = input<string>("Counter");
  count = input<number>(0);

  // Signal Output
  countChange = output<number>();

  increment() {
    // Emit new updated count back up to parent
    this.countChange.emit(this.count() + 1);
  }
}
```

#### **Parent Consumer Component (`dashboard.component.ts`):**

```typescript
import { Component, signal } from "@angular/core";
import { CustomCounterComponent } from "./custom-counter.component";

@Component({
  selector: "app-dashboard",
  standalone: true,
  imports: [CustomCounterComponent],
  template: `
    <h2>Dashboard View</h2>
    <app-custom-counter
      [label]="'Product Inventory Counter'"
      [count]="stockCount()"
      (countChange)="onStockUpdate($event)"
    >
    </app-custom-counter>
  `,
})
export class DashboardComponent {
  stockCount = signal<number>(10);

  onStockUpdate(newVal: number) {
    this.stockCount.set(newVal);
  }
}
```
