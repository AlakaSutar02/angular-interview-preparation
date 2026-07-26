## Q. What are Interceptors?

In Angular, an **Interceptor** is a mechanism that intercepts incoming HTTP requests and outgoing responses processed by Angular's `HttpClient`.

Think of it as **middleware for network requests**. It allows you to inspect, transform, or modify HTTP calls globally before they are sent to the server, or before their responses reach your components/services.

---

### **Primary Use Cases**

1. **Authentication & Authorization:** Automatically attach JWT bearer tokens to outgoing `Authorization` headers.
2. **Global Error Handling:** Intercept 401 (Unauthorized), 403 (Forbidden), or 500 (Server Error) status codes globally to show notifications or redirect to login.
3. **Logging & Analytics:** Log performance metrics (e.g., request duration) or log network failures to an internal server.
4. **Loading Spinners:** Automatically increment/decrement an active HTTP request counter to display a global loading spinner.
5. **Caching:** Return cached responses directly for repeated GET requests to avoid unnecessary network calls.

---

### **Modern Angular Example (Functional Interceptors - Angular 15+)**

In modern Angular (v15+), functional interceptors using `HttpInterceptorFn` are preferred over legacy class-based interceptors (`HttpInterceptor`).

#### **1. Creating a Functional Auth Interceptor (`auth.interceptor.ts`)**

```typescript
import { HttpInterceptorFn } from "@angular/common/http";
import { inject } from "@angular/core";
import { AuthService } from "./auth.service";

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  // Clone request because HttpRequest objects in Angular are immutable
  if (token) {
    const clonedRequest = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`,
      },
    });
    return next(clonedRequest);
  }

  // Pass request untransformed if no token exists
  return next(req);
};
```

#### **2. Registering Interceptors in `app.config.ts**`

```typescript
import { ApplicationConfig } from "@angular/core";
import { provideHttpClient, withInterceptors } from "@angular/common/http";
import { authInterceptor } from "./auth.interceptor";

export const appConfig: ApplicationConfig = {
  providers: [
    // Registers HTTP client with the functional interceptor chain
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
};
```

---

### **Key Rules to Remember in Interviews**

- **Immutability:** `HttpRequest` and `HttpResponse` objects are immutable. You cannot modify them directly; you must clone them using `req.clone()`.
- **Chaining:** Interceptors execute in the exact order they are declared in the `withInterceptors([...])` array for outgoing requests, and in **reverse order** for incoming responses.

## Q. Difference between setValue() and patchValue()

In Angular Reactive Forms, both **`setValue()`** and **`patchValue()`** are used to update the values of a `FormGroup` or `FormArray`, but they differ in strictness and how they enforce the form structure.

---

### **Key Difference Summary**

| Feature                   | `setValue()`                                                               | `patchValue()`                                                           |
| ------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Structure Enforcement** | **Strict.** The object must match the exact shape of the `FormGroup`.      | **Flexible.** Accepts partial objects and ignores missing properties.    |
| **Missing Control Error** | **Throws an error** if any control key is missing from the payload.        | **Fails silently** (updates available keys and ignores missing ones).    |
| **Extra Control Error**   | **Throws an error** if the payload contains key names not in the form.     | Ignores any extra keys not present in the `FormGroup`.                   |
| **Primary Use Case**      | Whole form resets/resets from API endpoints where structure is guaranteed. | Partial updates, single field updates, or dynamic/optional API payloads. |

---

### **Code Example Comparison**

Assume we have the following `FormGroup`:

```typescript
this.userForm = new FormGroup({
  name: new FormControl(""),
  email: new FormControl(""),
  age: new FormControl(null),
});
```

#### **1. Using `setValue()` (Strict)**

Must pass values for **all** form controls (`name`, `email`, and `age`):

```typescript
// ✅ WORKS: All keys are present
this.userForm.setValue({
  name: "John Doe",
  email: "john@example.com",
  age: 30,
});

// ❌ THROWS RUNTIME ERROR: "Must supply a value for form control with name: 'age'."
this.userForm.setValue({
  name: "John Doe",
  email: "john@example.com",
});
```

#### **2. Using `patchValue()` (Flexible)**

Can pass partial updates without throwing errors:

```typescript
// ✅ WORKS: Only updates 'email', leaves 'name' and 'age' unchanged
this.userForm.patchValue({
  email: "newemail@example.com",
});

// ✅ WORKS: Extra keys like 'address' are safely ignored
this.userForm.patchValue({
  name: "Jane Doe",
  address: "123 Main St",
});
```

---

> "The main difference between `setValue()` and `patchValue()` in Reactive Forms is strictness:
>
> - **`setValue()`** is strict. It requires you to supply an object that matches the exact structure of the `FormGroup`. If any control key is missing or extra, Angular throws a runtime error. It’s ideal when you want to enforce strict schema validation—for instance, when populating a form with a complete data model from a backend API.
> - **`patchValue()`** is flexible. It allows partial updates, modifying only the specified controls while leaving the rest unchanged without throwing errors. It’s best suited for single-field updates, dynamic forms, or multi-step form workflows."

Here is a set of senior-level, architect-ready responses and practical code implementations for each of these questions:

---

### **1) Core vs. Shared Modules: Differences & Practical Examples**

- **Core Module (`CoreModule`):**
- **Purpose:** Houses **singleton services**, global configurations, interceptors, guards, and app-wide single-instance layout components (e.g., `NavbarComponent`, `FooterComponent`, `AuthService`).
- **Rule:** Imported **ONLY ONCE** in the root `AppModule` (or registered at `app.config.ts` root). Importing it elsewhere creates duplicate singleton instances or breaks dependency isolation.

- **Shared Module (`SharedModule`):**
- **Purpose:** Houses **reusable UI artifacts**—common components (buttons, cards, modals), custom directives, and pipes—as well as re-exported common modules (`CommonModule`, `FormsModule`).
- **Rule:** Imported into **multiple lazy-loaded feature modules** wherever those UI components are needed. **Must NOT contain singleton services.**

#### **Practical Example Structure:**

```typescript
// core/core.module.ts
@NgModule({
  imports: [CommonModule, HttpClientModule],
  providers: [
    AuthService,
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
  ],
})
export class CoreModule {
  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    // Guard against importing CoreModule in feature modules
    if (parentModule) {
      throw new Error(
        "CoreModule is already loaded. Import it only in AppModule.",
      );
    }
  }
}

// shared/shared.module.ts
@NgModule({
  declarations: [CustomButtonComponent, HighlightDirective, CardMaskPipe],
  imports: [CommonModule, ReactiveFormsModule],
  exports: [
    CustomButtonComponent,
    HighlightDirective,
    CardMaskPipe,
    CommonModule,
    ReactiveFormsModule,
  ],
})
export class SharedModule {}
```

---

### **2) Practical Example of the `fromEvent` Operator**

`fromEvent` turns standard DOM events, Node EventEmitter events, or WebSocket events into an RxJS Observable stream.

#### **Real-World Use Case: Real-time Window Resize Listener / Search Typing**

```typescript
import {
  Component,
  ElementRef,
  AfterViewInit,
  ViewChild,
  inject,
  OnDestroy,
} from "@angular/core";
import { fromEvent, Subscription } from "rxjs";
import { debounceTime, distinctUntilChanged, map } from "rxjs/operators";

@Component({
  selector: "app-search",
  standalone: true,
  template: `<input
    #searchInput
    type="text"
    placeholder="Search products..."
  />`,
})
export class SearchComponent implements AfterViewInit, OnDestroy {
  @ViewChild("searchInput") searchInput!: ElementRef<HTMLInputElement>;
  private sub!: Subscription;

  ngAfterViewInit() {
    // Create an RxJS stream directly from DOM keyup events
    this.sub = fromEvent<KeyboardEvent>(this.searchInput.nativeElement, "keyup")
      .pipe(
        map((event) => (event.target as HTMLInputElement).value),
        debounceTime(400),
        distinctUntilChanged(),
      )
      .subscribe((searchTerm) => {
        console.log("Debounced Search API Triggered:", searchTerm);
      });
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
  }
}
```

---

### **3) Pure vs. Impure Pipes: Industry Use Cases**

- **Pure Pipes (Default):**
- **Behavior:** Angular executes a pure pipe **ONLY when it detects a change in primitive input values** (string, number, boolean) or a **new object reference**.
- **Industry Use Case:** High-performance data transformations like currency conversion, localized date formatting, or string masking (e.g., masking credit card digits).
- _Code:_ `@Pipe({ name: 'maskCard', pure: true })`

- **Impure Pipes (`pure: false`):**
- **Behavior:** Angular executes an impure pipe on **EVERY single Change Detection cycle**, regardless of whether the input reference changed.
- **Industry Use Case:** Filtering or sorting an array where items are mutated in-place without changing the array reference, or reactive state pipes that read internal component state.
- **Performance Warning:** Use impure pipes cautiously; heavy computations inside an impure pipe cause severe lag.

---

### **4) Access Token vs. Refresh Token**

| Feature             | Access Token                                               | Refresh Token                                                                                   |
| ------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Primary Purpose** | Grants access to protected API resources.                  | Requests a new Access Token when the current one expires.                                       |
| **Lifetime**        | Short-lived (e.g., 15 minutes to 1 hour).                  | Long-lived (e.g., 7 days to 30 days).                                                           |
| **Storage**         | Memory or secure HTTP-only cookies.                        | Strictly **`HttpOnly`, `Secure`, `SameSite` Cookie** (inaccessible to JS to prevent XSS theft). |
| **Transmission**    | Sent in every HTTP `Authorization: Bearer <token>` header. | Sent only to the `/auth/refresh` endpoint.                                                      |

---

### **5) How to Improve Angular Application Performance**

1. **Lazy Loading & Code Splitting:** Defer non-critical routes using `loadComponent` or `loadChildren`.
2. **`OnPush` Change Detection Strategy:** Eliminate unnecessary dirty-checking cycles across static component subtrees.
3. **Deferrable Views (`@defer`):** Load heavy components (e.g., charts, complex editors) lazily when scrolled into viewport.
4. **Virtual Scrolling (`CdkVirtualScrollViewport`):** Render only DOM elements visible in the viewport when managing large arrays (1,000+ items).
5. **Optimize Loops:** Use `@for (item of items; track item.id)` or `trackBy` in `*ngFor` to prevent DOM element re-creation.
6. **Image Optimization:** Use `NgOptimizedImage` for automated lazy loading, sizing, and WebP formatting.
7. **Zoneless / Signals:** Adopt Signals or `provideZonelessChangeDetection()` (Angular 18+) to bypass global `Zone.js` overhead.

---

### **6) What is Angular Universal (Angular SSR) and Why is it Used?**

**Angular Universal** (now integrated directly into `@angular/ssr`) renders Angular templates on a Node.js server ahead of time, returning pre-rendered, fully hydrated HTML to the client browser.

#### **Why it is used:**

1. **SEO Optimization:** Allows web crawlers and social media bots (Google, LinkedIn, Twitter) to parse complete HTML content rather than an empty `<app-root></app-root>`.
2. **Improved First Contentful Paint (FCP):** Displays visible HTML content instantly while JavaScript bundles download in the background.
3. **Low-Powered Mobile Devices:** Reduces client CPU workload during initial page load.

---

### **7) Dynamically Loading a Component in Angular**

Using **`ViewContainerRef`** and modern standalone component dynamic imports:

```typescript
import { Component, ViewChild, ViewContainerRef } from "@angular/core";

@Component({
  selector: "app-host",
  standalone: true,
  template: `
    <button (click)="loadModal()">Open Dynamic Modal</button>
    <ng-container #modalContainer></ng-container>
  `,
})
export class HostComponent {
  @ViewChild("modalContainer", { read: ViewContainerRef })
  container!: ViewContainerRef;

  async loadModal() {
    this.container.clear(); // Clear existing views
    // Dynamic ES module import
    const { ConfirmModalComponent } = await import("./confirm-modal.component");

    // Create and inject component dynamically
    const componentRef = this.container.createComponent(ConfirmModalComponent);
    componentRef.instance.title = "Delete Account?"; // Pass inputs
    componentRef.instance.confirmed.subscribe(() => this.container.clear()); // Handle outputs
  }
}
```

---

### **8) How to Create a Custom Pipe in Angular**

Here is a custom pipe that truncates long strings:

```typescript
import { Pipe, PipeTransform } from "@angular/core";

@Pipe({
  name: "truncate",
  standalone: true,
  pure: true,
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 20, trail: string = "..."): string {
    if (!value) return "";
    if (value.length <= limit) return value;
    return value.substring(0, limit) + trail;
  }
}

// Usage in Template:
// {{ longProductDescription | truncate:50:' [read more]' }}
```

---

### **9) The Role of Injector in Angular**

The **Injector** is the core mechanism of Angular's **Dependency Injection (DI)** framework.

- **Role:** It acts as a registry and factory for dependency instances. When a component or service requests a dependency in its constructor or via `inject()`, the Injector checks its container:
- If an instance already exists, it returns the **singleton instance**.
- If it does not exist, it instantiates the dependency, caches it, and returns it.

- **Hierarchical Injector Tree:** Angular uses an injector hierarchy:

1. **NullInjector:** Throws an error if a token is missing (unless `@Optional()`).
2. **EnvironmentInjector:** Holds root-level providers (`providedIn: 'root'`).
3. **ElementInjector:** Created per component/directive using `providers: [...]` or `viewProviders: [...]`.

---

### **10) Angular Guards and Their Types**

Route Guards are functional handlers that protect routing access:

1. **`canActivate`:** Controls if a user can visit a route.
2. **`canActivateChild`:** Controls if a user can visit child routes under a parent.
3. **`canDeactivate`:** Warns users before navigating away (e.g., unsaved form alert).
4. **`canMatch` (Replaced `canLoad`):** Controls if a route definition matches and whether its lazy module/component bundle should even be downloaded.

```typescript
// Modern Functional Guard Example
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.isLoggedIn()
    ? true
    : router.createUrlTree(["/login"], {
        queryParams: { returnUrl: state.url },
      });
};
```

---

### **11) What is `trackBy` in `ngFor` and Why is it Important?**

- **Problem without `trackBy`:** When array items change, Angular destroys and re-creates all DOM elements in the list, causing layout thrashing and slow re-renders.
- **Solution:** `trackBy` tells Angular how to uniquely identify each item (e.g., by `id`). Angular then reorders or updates only the specific changed DOM node rather than re-rendering the entire list.

```typescript
// Component:
trackById(index: number, item: { id: string }) { return item.id; }

```

```html
<!-- Template (Legacy) -->
<div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>

<!-- Modern Angular Control Flow (Built-in tracking requirement) -->
@for (item of items; track item.id) {
<div>{{ item.name }}</div>
}
```

---

### **12) What is `ngZone` in Angular?**

`NgZone` is an execution context wrapper around `Zone.js`. It isolates Angular’s change detection from third-party asynchronous operations.

- **Running Outside Angular:** When executing high-frequency events (like mouse movements, canvas drawing, or WebSocket listeners) that do **not** require UI updates, run them outside Angular to avoid triggering unnecessary global change detection cycles.

```typescript
import { Component, NgZone, ElementRef, inject } from "@angular/core";

@Component({ selector: "app-canvas", standalone: true, template: `` })
export class CanvasComponent {
  private ngZone = inject(NgZone);

  setupHighFrequencyListener() {
    this.ngZone.runOutsideAngular(() => {
      // Runs without triggering Angular Change Detection
      window.addEventListener("mousemove", (e) => {
        this.updateCanvasPosition(e.clientX, e.clientY);
      });
    });
  }

  updateCanvasPosition(x: number, y: number) {
    /* Fast DOM updates */
  }
}
```

---

### **13) Angular Lifecycle Hooks Real Usage**

1. `ngOnChanges`: Reacting to incoming `@Input()` value mutations.
2. `ngOnInit`: Making initial API calls, setting up reactive form structures.
3. `ngAfterViewInit`: Manipulating raw child DOM nodes via `@ViewChild()`.
4. `ngOnDestroy`: Cleaning up subscriptions, closing WebSockets, stopping `setInterval` timers.

---

### **14) Difference Between Template-Driven and Reactive Forms**

| Feature            | Template-Driven Forms                                | Reactive Forms                                                |
| ------------------ | ---------------------------------------------------- | ------------------------------------------------------------- |
| **Setup Location** | Defined mainly in the HTML template (`[(ngModel)]`). | Defined in the TypeScript class (`FormGroup`, `FormControl`). |
| **Data Flow**      | Asynchronous two-way data binding.                   | Synchronous observable-based data flow.                       |
| **Validation**     | Directive attributes in HTML (`required`, `email`).  | Code-based validator functions (`Validators.required`).       |
| **Testing**        | Requires DOM rendering testing (`TestBed`).          | Simple unit testing purely via class logic.                   |

---

### **15) Angular Change Detection Mechanism**

Change Detection syncs the application state with the DOM UI view:

1. **Trigger:** `Zone.js` intercepts async events (timers, HTTP responses, user clicks) and notifies Angular.
2. **Traversal:** Angular starts at the Root Component and traverses down the component tree top-to-bottom.
3. **Dirty Checking:** Compares template expression values against prior execution values.
4. **DOM Update:** If a value differs, Angular updates the corresponding DOM property.

---

### **16) Purpose of `@Input()` and `@Output()` Decorators**

- **`@Input()` / `input()`:** Receives data passed from a parent component down to a child component.
- **`@Output()` / `output()`:** Emits custom events from a child component back up to a parent component using `EventEmitter`.

---

### **17) Hot vs. Cold Observables**

- **Cold Observables:** Produces data **inside** the observable execution. It does not start emitting values until a subscriber registers (`.subscribe()`). Every subscriber gets its own independent data execution stream (e.g., standard `HttpClient.get()` calls).
- **Hot Observables:** Produces data **outside** the observable. It emits values regardless of whether subscribers exist. Multiple subscribers share the same execution stream and receive emissions simultaneously (e.g., `fromEvent`, `Subject`, `BehaviorSubject`).

---

### **18) Angular Service with CRUD & Generic Reactive Form Code**

#### **Service with CRUD Operations (`user.service.ts`)**

```typescript
import { Injectable, inject } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";

export interface User {
  id?: number;
  name: string;
  email: string;
}

@Injectable({ providedIn: "root" })
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = "https://api.example.com/users";

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  createUser(user: User): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }

  updateUser(id: number, user: User): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/${id}`, user);
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

#### **Generic Reactive Form with Validations (`user-form.component.ts`)**

```typescript
import { Component, inject, OnInit } from "@angular/core";
import {
  FormBuilder,
  FormGroup,
  ReactiveFormsModule,
  Validators,
} from "@angular/forms";

@Component({
  selector: "app-user-form",
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <div>
        <label>Name:</label>
        <input type="text" formControlName="name" />
        @if (isFieldInvalid("name")) {
          <span class="error">Name is required (min 3 chars).</span>
        }
      </div>

      <div>
        <label>Email:</label>
        <input type="email" formControlName="email" />
        @if (isFieldInvalid("email")) {
          <span class="error">Valid email is required.</span>
        }
      </div>

      <button type="submit" [disabled]="userForm.invalid">Submit</button>
    </form>
  `,
})
export class UserFormComponent implements OnInit {
  private fb = inject(FormBuilder);
  userForm!: FormGroup;

  ngOnInit() {
    this.userForm = this.fb.group({
      name: ["", [Validators.required, Validators.minLength(3)]],
      email: ["", [Validators.required, Validators.email]],
    });
  }

  isFieldInvalid(fieldName: string): boolean {
    const control = this.userForm.get(fieldName);
    return !!(control && control.invalid && (control.dirty || control.touched));
  }

  onSubmit() {
    if (this.userForm.valid) {
      console.log("Form Payload:", this.userForm.value);
    }
  }
}
```

---

### **19) Can We Write `[ngIf]`?**

- **Yes, absolutely.** Writing `[ngIf]` using square brackets converts the directive into a standard property binding.
- **How it works:** Instead of the structural dynamic template expander micro-syntax (`*ngIf="condition"`), you must explicitly apply `[ngIf]` on an `<ng-template>` element:

```html
<!-- Structural micro-syntax equivalent -->
<div *ngIf="isLoggedIn">Welcome Back</div>

<!-- Explicit [ngIf] property binding on ng-template -->
<ng-template [ngIf]="isLoggedIn">
  <div>Welcome Back</div>
</ng-template>
```

---

### **20) Difference Between `constructor` and `ngOnInit**`

- **`constructor()`:** Standard TypeScript class engine constructor. Executed when the class is instantiated. Reserved strictly for Dependency Injection. Inputs (`@Input()`) are **undefined** here.
- **`ngOnInit()`:** Angular lifecycle hook called once after component inputs have been bound. Safe place for component initialization, subscriptions, and API data fetching.

---

### **21) How to Secure Angular Applications From XSS Attacks**

1. **Automatic HTML Escaping:** Rely on Angular interpolation (`{{ content }}`), which escapes HTML entities automatically.
2. **Avoid Bypass APIs:** Avoid calling `DomSanitizer.bypassSecurityTrustHtml()` unless strictly cleansed via a library like DOMPurify.
3. **Avoid Raw Element References:** Do not write directly to `elementRef.nativeElement.innerHTML`. Use `Renderer2.setProperty()` instead.
4. **Strict CSP:** Implement strict HTTP `Content-Security-Policy` headers on the server side.

---

### **22) Explain RxJS Operators**

Operators are pure functions used to compose, transform, filter, and handle error streams within `.pipe()`:

- **Creation:** `of`, `from`, `fromEvent`, `interval`.
- **Transformation:** `map`, `concatMap`, `mergeMap`, `switchMap`, `exhaustMap`.
- **Filtering:** `filter`, `debounceTime`, `distinctUntilChanged`, `take`, `takeUntilDestroyed`.
- **Combination:** `forkJoin`, `combineLatest`, `zip`, `concat`.
- **Error Handling:** `catchError`, `retry`, `throwError`.

---

### **23) Difference Between `@ViewChild` and `@ContentChild**`

- **`@ViewChild()`:** Queries an element, directive, or child component located **inside the component's own template layout**. Available inside `ngAfterViewInit`.
- **`@ContentChild()`:** Queries an element or component that was projected into this component via **Content Projection (`<ng-content>`)**. Available inside `ngAfterContentInit`.

---

### **24) How Would You Debug an Angular Application?**

1. **Angular DevTools Browser Extension:** Inspect component tree structures, Signal values, performance profilers, and Change Detection execution costs.
2. **Source Maps in Chrome DevTools:** Set breakpoints directly in original TypeScript source code files (`CTRL+P`).
3. **RxJS Debugging:** Use `tap(console.log)` operators in pipeline streams or `rxjs-spy`.
4. **Network Panel:** Audit payload shapes, CORS blockages, and token headers.

---

### **25) Scenario-Based RxJS Operator Syntax Examples**

```typescript
import { HttpClient } from "@angular/common/http";
import { inject, Injectable } from "@angular/core";
import {
  concatMap,
  exhaustMap,
  forkJoin,
  mergeMap,
  Observable,
  switchMap,
} from "rxjs";

@Injectable({ providedIn: "root" })
export class RxScenarioService {
  private http = inject(HttpClient);

  // Scenario 1: Search Auto-complete (Cancel stale pending calls)
  search(query$: Observable<string>) {
    return query$.pipe(switchMap((q) => this.http.get(`/api/search?q=${q}`)));
  }

  // Scenario 2: Save items sequentially in order (Queue execution)
  saveInSequence(items$: Observable<any[]>) {
    return items$.pipe(concatMap((item) => this.http.post("/api/save", item)));
  }

  // Scenario 3: Fetch related items concurrently (Parallel processing)
  fetchParallel(ids$: Observable<number[]>) {
    return ids$.pipe(
      mergeMap((ids) => ids.map((id) => this.http.get(`/api/details/${id}`))),
    );
  }

  // Scenario 4: Load initial dashboard (Wait for all independent requests to complete)
  loadDashboard() {
    return forkJoin({
      profile: this.http.get("/api/profile"),
      settings: this.http.get("/api/settings"),
    });
  }
}
```

---

### **26) Code Where API Fails the First Time and Succeeds the Second Time**

Using RxJS `retry()` operator with conditional logic or mock state delay handling:

```typescript
import { HttpClient } from "@angular/common/http";
import { inject, Injectable } from "@angular/core";
import { Observable, of, throwError, timer } from "rxjs";
import { catchError, concatMap, retry } from "rxjs/operators";

@Injectable({ providedIn: "root" })
export class ResilienceService {
  private http = inject(HttpClient);

  fetchWithAutoRetry(): Observable<any> {
    let attemptCount = 0;

    // Simulated API call that fails on attempt 1, succeeds on attempt 2
    const simulatedApiCall$ = new Observable((subscriber) => {
      attemptCount++;
      if (attemptCount === 1) {
        console.warn("Attempt 1: Simulating Network Failure (500)...");
        subscriber.error(new Error("Internal Server Error"));
      } else {
        console.log("Attempt 2: API Success!");
        subscriber.next({ status: 200, data: "Success Payload" });
        subscriber.complete();
      }
    });

    return simulatedApiCall$.pipe(
      retry({
        count: 2, // Retries up to 2 times
        delay: (error, retryIndex) => {
          console.log(`Retry attempt #${retryIndex} initiated...`);
          return timer(1000); // Waits 1 second before retrying
        },
      }),
      catchError((err) => {
        console.error("All retries exhausted:", err);
        return throwError(() => err);
      }),
    );
  }
}
```
