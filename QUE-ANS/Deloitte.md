Here are clear, senior-level interview responses for each of your questions:

---

### **Core JavaScript & CSS**

#### **1) Difference between `map()` and `forEach()**`

- **`map()`:** Returns a **brand-new array** populated with the results of calling a provided function on every element. It does not mutate the original array and is chainable.
- **`forEach()`:** Iterates over the array and executes a callback function for each element. It returns `undefined` and is used purely for side effects (e.g., logging, modifying external variables).

#### **2) Difference between `var`, `let`, and `const**`

- **`var`:** Function-scoped, can be re-declared and re-assigned, hoisted and initialized with `undefined`.
- **`let`:** Block-scoped (`{}`), cannot be re-declared within the same scope, can be re-assigned, hoisted but stays in the **Temporal Dead Zone (TDZ)** until declared.
- **`const`:** Block-scoped, cannot be re-declared or re-assigned (though mutated objects/arrays remain mutable), hoisted into the TDZ.

#### **3) Explain Hoisting**

Hoisting is JavaScript's default behavior of moving variable and function declarations to the top of their containing scope during the compilation phase before code execution.

- Function declarations are fully hoisted and can be called before their definition.
- `var` variables are hoisted and initialized as `undefined`.
- `let` and `const` are hoisted but remain uninitialized in the **Temporal Dead Zone (TDZ)**; accessing them before initialization throws a `ReferenceError`.

#### **19) Explain CSS Box Model**

Every HTML element is represented as a rectangular box consisting of four layers (from inside out):

1. **Content:** The actual content (text, image, sub-elements).
2. **Padding:** Transparent area wrapping the content inside the border.
3. **Border:** Line surrounding the padding and content.
4. **Margin:** Transparent area outside the border separating the element from neighbors.

> _Note:_ Setting `box-sizing: border-box` ensures `width` and `height` include padding and border, preventing layout sizing bugs.

#### **20) Difference between `px`, `em`, and `rem**`

- **`px` (Pixel):** Absolute unit; remains fixed regardless of user or parent browser settings.
- **`em`:** Relative unit; calculated relative to the `font-size` of its **parent element** (or the element itself for non-font properties).
- **`rem` (Root EM):** Relative unit; calculated strictly relative to the `font-size` of the **root `<html>` element** (default 16px). Preferred for responsive typography and accessible design.

---

### **Angular Core & Architecture**

#### **4) Angular Building Blocks**

1. **Components:** UI views controlled by TypeScript classes with templates and styles.
2. **Templates:** HTML structure enhanced with Angular directives, bindings, and control flow.
3. **Directives:** Instructions that attach behavior or transform DOM elements (`Attribute` vs `Structural`).
4. **Services & Dependency Injection:** Reusable business logic injected across components via DI tokens.
5. **Pipes:** Functions used in templates to format and transform display data.
6. **Modules / Standalone API:** Configuration wrappers (legacy `NgModule` or modern Standalone components/providers).

#### **5) Difference between `constructor` and `ngOnInit**`

- **`constructor()`:** Standard ES6 class constructor. Called by the JavaScript engine when instantiating the class. Used strictly for Dependency Injection (DI) assignments. Input properties (`@Input`) are **not available** here yet.
- **`ngOnInit()`:** Angular lifecycle hook called once after Angular has fully initialized component inputs. Used for component initialization logic, API calls, and reading `@Input()` data.

#### **6) Explain Angular Lifecycle Hooks (In Execution Order)**

1. `ngOnChanges()`: Runs when data-bound `@Input()` properties change.
2. `ngOnInit()`: Runs once after first `ngOnChanges`.
3. `ngDoCheck()`: Custom change detection run on every change detection cycle.
4. `ngAfterContentInit()`: Runs once after projected content (`<ng-content>`) is initialized.
5. `ngAfterContentChecked()`: Runs after every check of projected content.
6. `ngAfterViewInit()`: Runs once after component view and child views are initialized.
7. `ngAfterViewChecked()`: Runs after every check of component and child views.
8. `ngOnDestroy()`: Runs immediately before Angular destroys the directive/component (cleanup subscriptions/timers).

#### **7) How components communicate with each other**

1. **Parent $\rightarrow$ Child:** `@Input()` bindings or Signal inputs (`input()`).
2. **Child $\rightarrow$ Parent:** `@Output()` with `EventEmitter` or Signal outputs (`output()`).
3. **Parent $\rightarrow$ Child View Access:** `@ViewChild()` / `viewChild()` or `@ContentChild()`.
4. **Unrelated / Sibling Components:** Shared RxJS `BehaviorSubject` Service, Signal Store, or NgRx.

#### **8) Major Changes from Angular 15 to Angular 19**

- **Angular 16:** Introduction of **Angular Signals** (`signal`, `computed`, `effect`), Hydration for SSR, and `takeUntilDestroyed`.
- **Angular 17:** New built-in **Control Flow syntax** (`@if`, `@for`, `@switch`), **Deferrable Views (`@defer`)**, and Standalone Components made default in `ng new`.
- **Angular 18:** **Event Hydration**, Zone-less change detection stability (`provideExperimentalZonelessChangeDetection`), and Signal Inputs/Outputs (`input()`, `output()`, `model()`).
- **Angular 19:** Incremental hydration, auto-imported standalone components, and full stability for Zoneless applications and Signals (`linkedSignal`).

#### **9) Why write API call logic in Service files instead of `component.ts`?**

- **Separation of Concerns:** Components focus solely on presentation logic; services handle data fetching and HTTP operations.
- **Reusability:** Multiple components can consume the same endpoint/data source.
- **Testability:** Easy to mock services in component unit tests without making HTTP network calls.
- **State Management & Caching:** Services can retain, transform, or cache API responses in memory across navigation.

#### **10) Folder Structure Setup for a New Project**

Adopt a **Domain-Driven Modular Architecture**:

```text
src/app/
├── core/              # Global singletons (Auth Guards, Interceptors, Core Services)
├── shared/            # Reusable UI components, directives, pipes, models
├── features/          # Domain feature modules (lazy-loaded routes)
│   ├── auth/
│   ├── dashboard/
│   └── user-profile/
├── app.config.ts      # Global providers
└── app.routes.ts      # Core routing setup

```

#### **11) What is Nx?**

**Nx** is a smart, fast, extensible build system and monorepo management framework. It provides advanced caching, visual dependency graphs, affected-build capabilities (only building/testing changed code), and seamless support for Micro Frontends and multi-project workspaces.

#### **12) Difference between Reactive Forms and Template-Driven Forms**

- **Reactive Forms:** Code-driven, explicit, immutable state model (`FormGroup`, `FormControl`). Built in component TS file. High scalability, synchronous data flow, and simple unit testing.
- **Template-Driven Forms:** Template-driven, implicit, mutable state model using two-way binding (`[(ngModel)]`). Built in HTML template. Best for simple forms; harder to unit test without DOM rendering.

#### **13) Difference between `setValue` and `patchValue**`

- **`setValue()`:** Strict update; requires exact match of all keys in the `FormGroup`. Throws an error if any control is missing or extra.
- **`patchValue()`:** Flexible update; accepts partial objects, updating specified keys while leaving rest untouched without throwing errors.

---

### **Signals**

#### **14) Explain Signals and their types**

Signals are reactive primitives tracking state access and dependencies across the application.

1. **Writable Signal (`signal(val)`):** Holds a value that can be mutated using `.set()` or `.update()`.
2. **Computed Signal (`computed(() => ...)`):** Read-only derived state; memoized and updates automatically when dependent signals change.
3. **Linked Signal (`linkedSignal(...)`):** A writable signal whose state is linked to a source signal but can be overridden manually.

#### **15) What is the use of `effect()` in Signals?**

An `effect()` schedules a side-effect operation that automatically re-executes whenever any Signal read inside its execution function changes.

#### **16) In which scenario to use `effect()` and when does it execute?**

- **Scenarios:** Logging/analytics tracking, syncing state with `localStorage`, or dynamic DOM operations (e.g., HTML Canvas rendering).
- **Execution:** Executes **at least once** asynchronously during the change detection rendering cycle, re-triggering whenever its signal dependencies emit new values.

> _Note:_ Do not use `effect()` to compute state transformations; use `computed()` instead.

---

### **Performance, Security & Features**

#### **17) Explain Debouncing in Angular and how to achieve it**

Debouncing delays function execution until a specified time interval passes without any new events firing (e.g., search typeaheads).

- **Implementation:** Use RxJS `debounceTime()` operator on a Reactive Form input `valueChanges`:

```typescript
this.searchControl.valueChanges
  .pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap((query) => this.apiService.search(query)),
  )
  .subscribe((results) => (this.results = results));
```

#### **18) How to check for Memory Leaks in an Angular application?**

1. **Chrome DevTools Memory Profiler:** Take a Heap Snapshot, perform actions (navigate away and back), take another snapshot, and compare allocations for un-destroyed Component instances.
2. **Performance Timeline:** Monitor JS heap growth over time using allocation instrumentation.
3. **Common Causes:** Unsubscribed RxJS observables, uncleaned `setInterval`/`setTimeout`, or unremoved DOM event listeners.

#### **21) How would you restrict access for different users? (Supervisor vs Operator)**

1. **Route Level:** Protect routes with a **`canActivate` Functional Guard** checking user roles retrieved from an `AuthService`.

```typescript
export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const user = authService.currentUser();
  return (
    user?.role === route.data["requiredRole"] || user?.role === "SUPERVISOR"
  );
};
```

2. **UI/Component Level:** Mask or hide UI actions using a structural directive or template signal check (`@if (auth.isSupervisor() || auth.isOwner(item.id))`).

#### **22) Can we create HTML elements from a custom directive?**

Yes. Inject **`Renderer2`** and **`ElementRef`** into the directive, and use `renderer.createElement()` along with `renderer.appendChild()` to attach child elements dynamically to the DOM host.

#### **23) Bank Card Number Masking Implementation**

Use a custom Angular Pipe:

```typescript
import { Pipe, PipeTransform } from "@angular/core";

@Pipe({
  name: "maskCard",
  standalone: true,
})
export class MaskCardPipe implements PipeTransform {
  transform(cardNumber: string): string {
    if (!cardNumber || cardNumber.length < 4) return cardNumber;
    const cleanNumber = cardNumber.replace(/\s+/g, "");
    const lastFour = cleanNumber.slice(-4);
    const maskedSection = "*".repeat(cleanNumber.length - 4);
    return `${maskedSection.match(/.{1,4}/g)?.join(" ") || maskedSection} ${lastFour}`;
  }
}
```

---

### **Testing & General Practices**

#### **24) Explain one challenge you faced in your project**

> _"In a recent project, we encountered severe UI frame drops when rendering a dashboard containing thousands of dynamic table rows with multiple input bindings. Every minor user edit triggered full-tree change detection cycles due to default strategy usage. I resolved this by migrating components to **`OnPush` Change Detection**, using **CDK Virtual Scrolling** to render only visible viewport rows, and replacing heavy template function calls with memoized Computed Signals."_

#### **25) Do you have experience writing test cases using Cypress or Jasmine/Karma?**

Yes, I write **Unit and Integration tests** using **Jasmine/Karma** (or Jest) with `TestBed` to test components, services, and interceptors, and **E2E tests** using **Cypress** (or Playwright) to test end-to-end user workflows like login, form submission, and route navigation.

#### **26) What is the need for writing test cases? Is it just for passing required coverage?**

No, test coverage is a secondary metric. The real purposes are:

1. **Preventing Regressions:** Ensuring new features don't break existing functionality.
2. **Refactoring Safety:** Confidently updating code structure or dependencies knowing tests will catch breakages.
3. **Living Documentation:** Tests explicitly define expected system behavior for future engineers.
4. **CI/CD Reliability:** Preventing broken code from reaching production via automated pipeline checks.

#### **27) Are you using any AI-assisted coding agents for daily development?**

Yes, I leverage AI tools like GitHub Copilot and ChatGPT for boilerplate generation, writing unit tests, debugging stack traces, and converting complex CSS/regex patterns—while maintaining human code review, security audits, and architectural oversight over all generated code.
