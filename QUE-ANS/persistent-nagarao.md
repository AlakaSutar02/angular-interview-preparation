Q. What are the new features of Angular 21?

1. Experimental Signal Forms: Angular 21 introduces a new reactive form API built on Signals.
2. Zoneless Change Detection by Default: Zone.js is no longer included by default in new Angular projects. Reduces bundle sizes by ~30 KB and eliminates unnecessary change detection cycles.
3. Angular Aria (Developer Preview)
4. Vitest as the Default Test Runner
5. Angular MCP Server (AI Integration)

Q. What are Directives in Angular?

Directives are TypeScript classes annotated with @Directive() that attach custom behavior, appearance, or structural transformations to elements in the DOM.
With modern Angular, directives are written as standalone by default.

Three Pillars of Directives:
1. Components: Directives with a template. Form the visual hierarchy and structure of the app.
2. Attribute: Directives modify the appearance, styling, or event-driven behavior of an existing DOM element/component.
   - Permission-based visibility ([appHasRole]), custom tooltips, auto-focus
3. Structural: Directives mutate DOM layout by adding, removing, or manipulating elements via ViewContainerRef and TemplateRef.
   - Conditional rendering, dynamic list rendering.

Historically, if a component needed the behavior of three different directives, you had to manually declare all of them on the host element.
To solve this in v15, Directive Composition API -- hostDirectives -- was introduced.
We can inject behaviors like a tracking directive, a tooltip directive, and a validation directive directly into a single host without polluting the template.

```typescript
@Directive({
  selector: '[hasPermission]',
  standalone: true
})
export class HasPermissionDirective {
  private templateRef = inject(TemplateRef);
  private viewContainer = inject(ViewContainerRef);
  private authService = inject(AuthService);

  permission = input.required<string | string[]>({ alias: 'hasPermission' });

  constructor() {
    effect(() => {
      const required = this.permission();
      const hasAccess = this.authService.hasPermission(required);

      this.viewContainer.clear();

      if (hasAccess) {
        this.viewContainer.createEmbeddedView(this.templateRef);
      }
    });
  }
}
```

```typescript
@Directive({
  selector: '[appTooltip]',
  standalone: true,
  host: {
    '[class.has-tooltip]': 'true',
    '(mouseenter)': 'showTooltip()',
    '(mouseleave)': 'hideTooltip()'
  }
})
export class TooltipDirective {
  showTooltip() { /* ... */ }
  hideTooltip() { /* ... */ }
}
```

Q. Explain Lifecycle Hooks in Angular in modern Angular.

---

## 1. High-Level Architectural Shift

Historically, Angular relied on class methods implementing interfaces (`OnInit`, `OnChanges`, `OnDestroy`) tied to `Zone.js` change detection.

In modern Angular:

- **Reactive inputs replaces `ngOnChanges`:** Signal inputs (`input()`, `computed()`) replace imperative input tracking.
- **Side-effects replace state syncing:** `effect()` handles logic when reactive state changes.
- **`DestroyRef` / `takeUntilDestroyed` replaces `ngOnDestroy`:** Explicit, injection-context cleanup primitives prevent memory leaks.
- **Zoneless compatibility:** Modern primitives work naturally without relying on `zone.js` global tick loops.

---

## 2. Legacy vs. Modern Angular Equivalents

| Lifecycle Phase | Legacy Class Hook | Modern Angular Paradigm (v16+) | Why the Shift? |
| --- | --- | --- | --- |
| **Input Changes** | `ngOnChanges(changes)` | `input()`, `computed()`, `effect()` | Eliminates manually parsing `SimpleChanges` objects; updates are fine-grained. |
| **Initialization** | `ngOnInit()` | `constructor` + `inject()` + Signal initializers | Signals are synchronous and initialized at instantiation time. |
| **Destruction** | `ngOnDestroy()` | `DestroyRef.onDestroy()` / `takeUntilDestroyed()` | Enables cleanup logic inside functions/services without requiring class inheritance. |
| **DOM / View Ready** | `ngAfterViewInit()` | `afterNextRender()` / `afterRender()` | Explicit SSR-safe APIs designed to run strictly on the browser client. |

---

## 3. Deep-Dive: Modern Angular Primitives

### A. Replacing `ngOnChanges`: Signals & `computed()`

In legacy Angular, reacting to changing `@Input()` properties required `ngOnChanges`:

```typescript
// ? Legacy Approach
@Input() userId!: string;
ngOnChanges(changes: SimpleChanges) {
  if (changes['userId']) {
    this.fetchUserData(this.userId);
  }
}

// ? Modern Angular Approach (Signal Inputs)
userId = input.required<string>();
userData = computed(() => this.userService.getUser(this.userId()));
```

### B. Replacing `ngOnDestroy`: `DestroyRef` and `takeUntilDestroyed`

In modern Angular, cleanup logic can be co-located directly where resources are initialized using `DestroyRef` or RxJS's `takeUntilDestroyed` operator.

```typescript
import { Component, inject, DestroyRef } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ ... })
export class AnalyticsComponent {
  private destroyRef = inject(DestroyRef);

  constructor() {
    this.eventStream$.pipe(
      takeUntilDestroyed()
    ).subscribe();

    const timer = setInterval(() => console.log('Ping'), 1000);
    this.destroyRef.onDestroy(() => clearInterval(timer));
  }
}
```

### C. Replacing `ngAfterViewInit`: `afterNextRender()` & `afterRender()`

`ngAfterViewInit` executes on both the server (SSR) and client, often causing DOM reference errors on Node.js runtimes. Modern Angular introduces explicitly scoped DOM hooks:

- `afterNextRender()`: Runs once after the next change detection cycle completes in the browser.
- `afterRender()`: Runs after every change detection cycle in the browser.

```typescript
import { Component, ElementRef, viewChild, afterNextRender } from '@angular/core';

@Component({ ... })
export class ChartComponent {
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('chartCanvas');

  constructor() {
    afterNextRender(() => {
      this.initializeThirdPartyChart(this.canvas().nativeElement);
    });
  }
}
```

---

## 4. Summary Table of Traditional Hooks (When still needed)

If working with legacy codebases or complex scenarios, traditional hooks are ordered as:

1. `ngOnChanges`: Triggered when `@Input` bindings update.
2. `ngOnInit`: Runs once after inputs are bound.
3. `ngDoCheck`: Custom change detection hook (avoid in modern apps due to performance cost).
4. `ngAfterContentInit` / `ngAfterContentChecked`: Triggered after `<ng-content>` project content is initialized/checked.
5. `ngAfterViewInit` / `ngAfterViewChecked`: Triggered after component view and child views are initialized/checked.
6. `ngOnDestroy`: Cleanup hook before DOM removal.

---

> *"In modern Angular, we are moving away from imperative class-level lifecycle hooks toward declarative reactive primitives. Instead of overriding `ngOnChanges` or `ngOnInit`, we use Signal inputs, `computed()` properties, and `effect()`. For DOM manipulation, instead of `ngAfterViewInit`, we use SSR-safe primitives like `afterNextRender()`. Finally, `DestroyRef` and `takeUntilDestroyed()` allow us to encapsulate lifecycle cleanup directly within composable functions or services, eliminating boilerplate and making our applications fully ready for Zoneless rendering."*

Q. Which RxJS operators have you used?

## 1. Higher-Order Flattening Operators (Race Conditions & Concurrency)

These operators manage inner observables (like HTTP requests) triggered by outer emissions.

### A. `switchMap` (The "Cancel & Switch" Operator)

- **Architectural Use Case:** Real-time search type-ahead / auto-complete inputs.
- **Why:** If the user types `A`, then `B`, then `C`, `switchMap` immediately unsubscribes from (and cancels) the pending in-flight HTTP requests for `A` and `B`, ensuring only the latest query `C` returns data to the UI.

```typescript
readonly searchControl = new FormControl('');

readonly searchResults$ = this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(searchTerm => this.apiService.searchProducts(searchTerm))
);
```

### B. `exhaustMap` (The "Ignore While Busy" Operator)

- **Architectural Use Case:** Form submission / payment checkout / login buttons.
- **Why:** Prevents duplicate network requests caused by users accidentally double-clicking or rapid-clicking a "Submit Order" button. It ignores all new emissions until the current active inner observable completes.

```typescript
readonly submitClick$ = new Subject<void>();

readonly submitResponse$ = this.submitClick$.pipe(
  exhaustMap(() => this.checkoutService.processPayment(this.cartData()))
);
```

### C. `concatMap` vs `mergeMap`

- `concatMap` (Sequential Queueing): Used for ordered operations, such as sequentially uploading files or executing batch database updates where sequence order matters.
- `mergeMap` (Parallel Execution): Used for concurrent, order-independent operations, such as executing multiple `DELETE` HTTP calls for selected items in a list as quickly as possible.

---

## 2. Combination Operators (Stream Orchestration)

### A. `combineLatest`

- **Architectural Use Case:** Multi-filter dashboards (e.g., Search Text + Category Filter + Date Range + Pagination).
- **Why:** Waits for all streams to emit at least once, then emits an array containing the latest value from every stream whenever any input stream updates.

```typescript
readonly filteredData$ = combineLatest([
  this.searchTerm$,
  this.selectedCategory$,
  this.pageIndex$
]).pipe(
  switchMap(([term, category, page]) =>
    this.apiService.fetchFilteredReport(term, category, page)
  )
);
```

### B. `forkJoin`

- **Architectural Use Case:** Initializing page metadata / loading lookup tables during app bootstrapping.
- **Why:** Operates like `Promise.all()`. It waits for multiple independent, finite observables (like HTTP GET calls) to complete, then emits their final values together as a single payload.

```typescript
loadPageData(): Observable<DashboardData> {
  return forkJoin({
    userProfile: this.userService.getProfile(),
    roles: this.authService.getRoles(),
    systemConfig: this.configService.getAppConfig()
  });
}
```

---

## 3. Rate-Limiting & Performance Tuning Operators

### A. `debounceTime` vs `throttleTime`

- `debounceTime(300)`: Delays emitting until a specified time window passes without any new emissions. Used on search input text boxes to avoid spamming the backend API on every keystroke.
- `throttleTime(1000)`: Emits the first value immediately, then ignores subsequent emissions for the specified duration. Used for window resizing listeners or scroll event listeners.

---

## 4. Error Handling & Resilience Operators

### A. `catchError` with `of()` Fallback

- **Architectural Use Case:** Preventing stream termination and providing graceful fallback state.
- **Why:** If an HTTP request inside an RxJS pipeline errors out, the entire stream completes unless intercepted by `catchError`. Returning `of(fallbackData)` ensures the UI continues functioning gracefully.

```typescript
readonly userMetrics$ = this.userService.getMetrics().pipe(
  catchError(error => {
    this.logger.logError(error);
    return of({ totalViews: 0, activeUsers: 0 });
  })
);
```

---

## 5. Architectural Bridge: RxJS + Angular Signals (`toSignal`)

In modern Angular (v16-v21), a Senior Lead handles RxJS in services and converts streams into **Signals** at the component layer:

```typescript
@Component({ ... })
export class ProductDashboardComponent {
  private productService = inject(ProductService);

  readonly products = toSignal(
    this.productService.products$.pipe(
      catchError(() => of([]))
    ),
    { initialValue: [] }
  );
}
```

---

> *"In my architectural experience, I select RxJS operators based on concurrency, timing, and error resiliency needs. I use `switchMap` for search type-aheads to cancel redundant in-flight API calls, `exhaustMap` on action buttons to prevent duplicate form submissions, and `combineLatest` for orchestrating dynamic multi-filter UI views. I handle global state or async HTTP pipelines with RxJS in the service layer, and seamlessly convert them to Signals using `toSignal` for clean, zoneless-ready UI templates."*

Q. How to use RxJS in TypeScript if you want to handle errors and retry API up to 3 times?

RxJS provides the modern `retry()` operator, which accepts a configuration object allowing you to specify count, delay, and conditional retry criteria.

```typescript
import { Observable, timer } from 'rxjs';
import { retry } from 'rxjs/operators';

/**
 * Custom operator for enterprise resilience with exponential backoff and randomized jitter
 */
export function retryWithBackoff(config: {
  maxRetries?: number;
  initialDelayMs?: number;
  backoffFactor?: number;
}) {
  const { maxRetries = 3, initialDelayMs = 1000, backoffFactor = 2 } = config;

  return <T>(source: Observable<T>): Observable<T> =>
    source.pipe(
      retry({
        count: maxRetries,
        delay: (error, retryCount) => {
          if (error.status === 401 || error.status === 403 || error.status === 422) {
            throw error;
          }

          const exponentialDelay = initialDelayMs * Math.pow(backoffFactor, retryCount - 1);
          const jitter = Math.random() * 500;
          const totalDelay = exponentialDelay + jitter;

          return timer(totalDelay);
        }
      })
    );
}

// Clean usage in API services:
// this.http.get('/api/resource').pipe(retryWithBackoff({ maxRetries: 3 }))
```

Q. After subscribing, will you handle error or before subscribing?

Handling errors inside the pipeline with operators like `catchError()` (before subscribing) offers critical architectural advantages: `catchError()` allows you to intercept an error, perform logging, and return a fallback stream or safe default value (using `of()`). This prevents the observable stream from prematurely terminating or breaking downstream subscribers.

In Modern Angular (v16-v21), the ultimate goal is to avoid manual `.subscribe()` calls completely:

- Service Layer: Build robust RxJS pipelines with `pipe(retry(), catchError())`.
- Component Layer: Convert the pipeline to a Signal using `toSignal()` or consume it via the `async` pipe.

```typescript
@Component({ ... })
export class UserListComponent {
  private userService = inject(UserService);
  private notificationService = inject(NotificationService);

  readonly users = toSignal(
    this.userService.getUsers().pipe(
      catchError(err => {
        this.notificationService.showError('Failed to load users');
        return of([]);
      })
    ),
    { initialValue: [] }
  );
}
```

Q. Explain complete flow -- how response comes, subscribe works, and error handling happens.

---

## The Execution Flow (Standard `GET` Request)

```text
[1. Pipeline Definition] --> Declarative setup (No network call yet)
                            |
[2. Subscription Trigger] --> `.subscribe()` or `toSignal()` kicks off request
                            |
                +-----------------------+
                ?                       ?
       [3A. SUCCESS PATH]         [3B. FAILURE PATH]
       (Server returns 200)       (Server returns 404/500)
                |                       |
                ?                       ?
   `HttpClient` parses JSON      `HttpClient` emits HttpErrorResponse
                |                       |
                ?                       ?
   `catchError()` passes through   `catchError()` intercepts error
                |                       +-- Logs error / telemetry
                ?                       +-- Returns `of(FALLBACK_DATA)`
   Subscriber `.next()`               |
   Receives valid response            ?
                |               Subscriber `.next()`
                ?               Receives safe fallback value
   Subscriber `.complete()`            |
   Stream closes cleanly               ?
                            Stream closes cleanly
```

---

### Step 1: Pipeline Assembly (Lazy Phase)

When you write the RxJS pipe, Angular constructs an execution plan. **No HTTP call is dispatched yet.**

```typescript
const users$ = this.http.get<User[]>('/api/users').pipe(
  catchError((error) => {
    this.logger.logError(error);
    return of([]); // Return fallback empty list
  })
);
```

### Step 2: Subscription (Trigger Phase)

The request is triggered when a subscriber listens to the stream (e.g., via `toSignal()`, `async` pipe, or `.subscribe()`):

1. Subscription flows **upstream** from consumer to `HttpClient`.
2. `HttpClient` instantiates the browser's `XMLHttpRequest` or `fetch()` API and sends the GET request over the network.

---

### Step 3A: Success Path (200 OK Response)

1. **Response Received:** The backend returns data with a `200 OK` status.
2. **Parsing:** `HttpClient` converts the JSON payload into TypeScript objects.
3. **Pipeline Pass-Through:** The `catchError()` operator sees a successful emission and **does nothing**--it simply forwards the data downstream.
4. **Consumer Notification:** The subscriber's `.next(data)` callback executes, updating component state or Signals.
5. **Stream Completion:** `HttpClient` emits `.complete()`, closing the stream and cleaning up resources automatically.

---

### Step 3B: Error Path (e.g., 404 or 500 Error Response)

1. **Error Triggered:** The server responds with an error status (e.g., `500 Internal Server Error`). `HttpClient` emits an `HttpErrorResponse` via the `.error()` notification channel.
2. **Interception:** `catchError()` intercepts the `.error()` notification before it hits the subscriber.
3. **Recovery:**
   - It logs the error to a monitoring service.
   - It executes `return of([])` (or fallback data).
   - **Crucial Detail:** `of()` transforms the stream's state **from an `.error()` back into a normal `.next()` emission**.
4. **Consumer Notification:** The subscriber receives the fallback value in its `.next()` callback (preventing UI crashes).
5. **Stream Completion:** Because the error was caught and resolved, the observable completes cleanly.

---

> *"For a standard GET request, defining `http.get().pipe(catchError())` builds a lazy pipeline that only fires when subscribed to. On a successful 200 OK, `HttpClient` passes data through `.next()` directly to the subscriber and then completes. On an HTTP error, `HttpClient` emits an error notification, which `catchError()` intercepts to log telemetry and emit a fallback value via `of()`. This converts the error into a valid `.next()` emission, keeping the UI stable without unhandled client-side exceptions."*

Instead of writing code inside the `.subscribe()` block, you chain RxJS operators within `.pipe()` to transform, filter, combine, or handle errors *before* data reaches the consumer.

---

## The 4 Core Categories of Pre-Subscription Logic

```text
   [ Raw Data Source (e.g., http.get) ]
                   |
                   ?
+-----------------------------------------+
|              .pipe(...)                 |
|                                         |
| 1. Transformation (map, scan)          |
| 2. Filtering      (filter, debounceTime)|
| 3. Error / Retry  (retry, catchError)   |
| 4. Side Effects   (tap for telemetry)   |
+-----------------------------------------+
                   |
                   ?
      [ Subscriber / Signal Consumer ]
```

---

### 1. Data Transformation Logic (`map`)

Shape, clean, or adapt raw backend DTOs into the exact view-model format the UI requires **before** emitting it to components.

```typescript
readonly userList$ = this.http.get<UserDto[]>('/api/users').pipe(
  map(users => users.map(user => ({
    id: user.id,
    fullName: `${user.firstName} ${user.lastName}`,
    isAdmin: user.roles.includes('ADMIN')
  })))
);
```

---

### 2. Side Effect & Telemetry Logic (`tap`)

Execute side-effects--such as triggering loading spinners, writing audit logs, or tracking analytics--without mutating the data payload passing through the stream.

```typescript
readonly orderDetails$ = this.http.get<Order>('/api/order/123').pipe(
  tap({
    subscribe: () => this.loadingService.show(),
    next: () => this.loadingService.hide(),
    error: (err) => {
      this.loadingService.hide();
      this.analytics.trackError('ORDER_FETCH_FAILED', err);
    }
  })
);
```

---

### 3. Rate-Limiting & Filtering Logic (`filter`, `debounceTime`, `distinctUntilChanged`)

Prevent unnecessary network requests or UI re-renders by enforcing business constraints upstream.

```typescript
readonly searchResults$ = this.searchControl.valueChanges.pipe(
  filter((term): term is string => !!term && term.trim().length >= 3),
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.apiService.search(term))
);
```

---

### 4. Error Resilience Logic (`retry`, `catchError`)

Intercept server failures, trigger retry policies, and return fallback data before the stream hits the UI layer.

```typescript
readonly dashboardData$ = this.http.get<Dashboard>('/api/dashboard').pipe(
  retry({ count: 2, delay: 1000 }),
  catchError((error) => {
    this.notification.showError('Could not refresh dashboard');
    return of(EMPTY_DASHBOARD_STATE);
  })
);
```

---

## Lead Architecture Takeaway: Converting to Signals

In modern Angular (v16-v21), you build all this logic declaratively in the service pipe, then convert it directly into a **Signal** using `toSignal()`. This eliminates `.subscribe()` boilerplate altogether while preserving fine-grained UI reactivity:

```typescript
@Component({ ... })
export class DashboardComponent {
  private dashboardService = inject(DashboardService);

  readonly dashboard = toSignal(
    this.dashboardService.dashboardData$,
    { initialValue: EMPTY_DASHBOARD_STATE }
  );
}
```

---

> *"We implement pre-subscription logic declaratively using RxJS operators inside `.pipe()`. We use `map` for DTO transformations, `tap` for side-effects like loaders or analytics, filtering operators to enforce business rules, and `catchError` with `retry` for error resilience. This keeps our service pipelines self-contained, testable, and clean--allowing us to expose streams safely to UI components via `toSignal()` without writing imperative code inside subscription blocks."*

Q. Why do we use `pipe` before subscribing?

`.pipe()` is used to declaratively assemble processing pipelines. It transforms, filters, and manages error recovery on the stream before data hits the consumer, keeping subscription callbacks clean and transformation logic reusable and testable.

Q. Then why do we use `map` operator?

`map` is used for synchronous data projection. It takes values emitted by the source observable, applies a transformation function, and emits the transformed result downstream before the consumer receives it.

Q. What does `filter` operator do?

The `filter` operator acts as a conditional gatekeeper for streams. It evaluates every item emitted by the source Observable against a predicate function.

- If the condition evaluates to true, the item passes through to the next operator or subscriber.
- If the condition evaluates to false, the item is dropped/ignored completely--it will never reach downstream operators or the subscriber.

Common enterprise use cases:
- Preventing null/undefined processing: Block null or undefined values from breaking downstream components.
- Min-length search triggers: Ignore search terms that have fewer than 3 characters.
- Role/state guards: Filter events based on specific status flags (e.g., only process orders where `status === 'COMPLETED'`).

Q. How will you implement lazy loading in Angular?

Lazy loading splits application bundles into smaller, on-demand JavaScript chunks. Instead of serving a massive initial `main.js` bundle to the user, Angular downloads only the critical initial code, deferring remaining routes and heavy components until requested.

Route-based lazy loading in modern standalone Angular applications (Angular 15-21+) uses `loadComponent` directly, eliminating the legacy need for `NgModule`-based `loadChildren`.

```typescript
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES)
  }
];
```

Q. What are ways to optimize Angular application?

---

## 1. Runtime & Change Detection (CPU & Memory)

### A. Zoneless & Signal-Based Architecture

- Strategy: Transition from `zone.js` global change detection to fine-grained **Signals** or `ChangeDetectionStrategy.OnPush`.
- Impact: Eliminates full component tree re-evaluations whenever an asynchronous event fires.
- Modern best practice: In Angular 21, run applications zoneless with `provideExperimentalZonelessChangeDetection()` so change detection only triggers for components with explicit signal or input mutations.

### B. Template Loop Tracking (`@for` with `track`)

- Strategy: Use modern `@for (item of items; track item.id)` syntax instead of legacy `*ngFor` without `trackBy`.
- Impact: Prevents Angular from destroying and re-creating DOM nodes when array references change, updating only modified items.

### C. Preventing Memory Leaks

- Strategy: Clean up infinite RxJS subscriptions using `takeUntilDestroyed()`, `DestroyRef`, or by piping observables directly into template Signals via `toSignal()`.

---

## 2. Bundle Size & Loading Strategy (Initial Load & Web Vitals)

### A. Dynamic Deferral (`@defer`) & Route-Based Lazy Loading

- Strategy: Lazy load routes using `loadComponent` and heavy template sections (charts, code editors, complex dialogs) using `@defer (on viewport)` or `@defer (on interaction)`.
- Impact: Drastically reduces initial JavaScript bundle size, improving Largest Contentful Paint (LCP) and Total Blocking Time (TBT).

### B. Tree-Shakable Services & Standalone Components

- Strategy: Use `standalone: true` components and `{ providedIn: 'root' }` services.
- Impact: Allows modern bundlers (Esbuild/Vite) to eliminate unused code paths during production builds.

---

## 3. Build & Infrastructure Optimization

### A. Ahead-of-Time (AOT) & Esbuild/Vite Builders

- Strategy: Ensure builds use the modern Angular Application Builder (`@angular-devkit/build-angular:application`), which uses Esbuild for fast optimized production bundles and Vite for local development HMR.

### B. Bundle Analysis

- Strategy: Audit bundle composition regularly using tools like `webpack-bundle-analyzer` or `ng build --stats-json` to detect duplicate dependencies or oversized third-party libraries.

---

> *"I approach Angular optimization across four pillars: **Runtime**, by leveraging Signals and Zoneless change detection to eliminate unneeded DOM re-renders; **Bundle Architecture**, using `@defer` blocks and lazy routes to minimize initial JS payload; **Build Tooling**, using the modern Esbuild application builder and dependency auditing; and **Network/Rendering**, using SSR non-destructive hydration and `NgOptimizedImage` to maximize Web Vitals like LCP and CLS."*

Q. Difference between Signals and BehaviorSubject.

The key difference between a Signal and a BehaviorSubject comes down to fine-grained state management versus stream processing.

- A `BehaviorSubject` is an asynchronous RxJS push stream that requires subscription management and relies on change detection sweeps to update views.
- A `Signal` is a synchronous, fine-grained reactive value primitive that tracks template dependencies directly.

In modern Angular, Signals are used as the primary mechanism for synchronous UI state because they provide glitch-free computed state and enable zoneless rendering, while RxJS is reserved for complex asynchronous operations like API search type-aheads or websocket streams.

Q. If I want to change CSS dynamically, how will you do it?

In Angular, the choice of dynamic styling depends on the scope. For component-level state changes, leverage reactive Signal bindings with `[class]` or `[style]`.

```html
<!-- Single class toggle -->
<div [class.is-active]="isActive()">Active State</div>

<!-- Multiple classes via object or Signal -->
<div [class]="statusClasses()">Status Banner</div>
```

```typescript
readonly status = signal<'success' | 'warning' | 'danger'>('success');

readonly statusClasses = computed(() => ({
  'bg-green-100 text-green-800': this.status() === 'success',
  'bg-yellow-100 text-yellow-800': this.status() === 'warning',
  'bg-red-100 text-red-800': this.status() === 'danger'
}));
```

For reusable behavior, build custom attribute directives:

```typescript
import {
  Directive,
  ElementRef,
  inject,
  input,
  Renderer2
} from '@angular/core';

@Directive({
  selector: '[appHoverHighlight]',
  standalone: true,
  host: {
    '(mouseenter)': 'onMouseEnter()',
    '(mouseleave)': 'onMouseLeave()',
    '(focusin)': 'onMouseEnter()',
    '(focusout)': 'onMouseLeave()'
  }
})
export class HoverHighlightDirective {
  private el = inject(ElementRef);
  private renderer = inject(Renderer2);

  hoverColor = input<string>('#f3f4f6', { alias: 'appHoverHighlight' });
  defaultColor = input<string>('transparent');

  protected onMouseEnter(): void {
    this.setBgColor(this.hoverColor());
  }

  protected onMouseLeave(): void {
    this.setBgColor(this.defaultColor());
  }

  private setBgColor(color: string): void {
    this.renderer.setStyle(this.el.nativeElement, 'background-color', color);
    this.renderer.setStyle(this.el.nativeElement, 'transition', 'background-color 0.2s ease-in-out');
  }
}
```

Q. I want to toggle element dynamically without using `*ngIf`. How will you do it?

To toggle an element without `*ngIf` or `@if`, choose based on DOM footprint:

- To keep state and preserve DOM nodes: use property bindings like `[class.hidden]`, `[hidden]`, or `[style.display]`.
- To completely detach/attach DOM nodes: write a custom structural directive using `ViewContainerRef` and `TemplateRef`, or load components dynamically using `ViewContainerRef.createComponent()`.

```typescript
import { Directive, inject, input, TemplateRef, ViewContainerRef, effect } from '@angular/core';

@Directive({
  selector: '[appRenderIf]',
  standalone: true
})
export class RenderIfDirective {
  private templateRef = inject(TemplateRef);
  private viewContainer = inject(ViewContainerRef);

  appRenderIf = input.required<boolean>();

  constructor() {
    effect(() => {
      this.viewContainer.clear();
      if (this.appRenderIf()) {
        this.viewContainer.createEmbeddedView(this.templateRef);
      }
    });
  }
}
```

```html
<!-- Custom structural directive syntax -->
<div *appRenderIf="isVisible()">
  <p>Element is dynamically added or removed from the DOM tree.</p>
</div>
```

Q. In a large Angular application, how do you pass data between components? Different ways to share data between components.

---

## Summary Matrix

| Scenario | Recommended Approach | Modern API (v16-v21+) | Legacy / Alternative API |
| --- | --- | --- | --- |
| **Parent ? Child** | Signal Inputs / Input Decorator | `input()` / `input.required()` | `@Input()` |
| **Child ? Parent** | Output Emitter | `output()` | `@Output() new EventEmitter()` |
| **Parent ? Deep Child** | Dependency Injection (DI) | `inject(ParentComponent)` | `@Host()` / `@Optional()` |
| **Unrelated / Cross-Tree** | Reactive Service / State | `signal()` / `WritableSignal` | `BehaviorSubject` / NgRx / Signal Store |
| **Route Navigation** | Router State & Resolver | `withComponentInputBinding()` | `ActivatedRoute.snapshot` |

---

## 1. Parent-to-Child Communication (Downstream Data Flow)

### Modern Signal Inputs (`input()`)

In modern Angular (v17+), input properties use **Signal Inputs**. They are read-only signals that allow fine-grained reactive tracking without requiring `ngOnChanges`.

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `<h3>{{ user().name }}</h3>`
})
export class UserCardComponent {
  user = input.required<User>();
  isEditable = input<boolean>(false);
}
```

---

## 2. Child-to-Parent Communication (Upstream Event Notification)

### Modern Output Function (`output()`)

Instead of legacy `EventEmitter`, modern Angular uses the lightweight `output()` API to emit custom events to parent listeners.

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `<button (click)="onDelete()">Delete</button>`
})
export class UserCardComponent {
  userId = input.required<string>();
  deleteUser = output<string>();

  onDelete() {
    this.deleteUser.emit(this.userId());
  }
}
```

---

## 3. Unrelated / Cross-Tree Components (Services & State Management)

For components located in completely different parts of the DOM tree, passing data through intermediate components ("prop drilling") is an anti-pattern. Instead, use **Injectable Services**.

### A. Modern Lightweight Signal Store (Service-per-Feature)

```typescript
@Injectable({ providedIn: 'root' })
export class CartStateService {
  private cartItemsSignal = signal<CartItem[]>([]);
  readonly cartItems = this.cartItemsSignal.asReadonly();

  readonly totalAmount = computed(() =>
    this.cartItems().reduce((sum, item) => sum + item.price, 0)
  );

  addItem(item: CartItem) {
    this.cartItemsSignal.update(items => [...items, item]);
  }
}
```

### B. RxJS Service Store (`BehaviorSubject`)

Used when state updates need to tie directly into complex RxJS operators (`switchMap`, `debounceTime`):

```typescript
@Injectable({ providedIn: 'root' })
export class SearchStateService {
  private searchQuerySubject = new BehaviorSubject<string>('');
  readonly searchQuery$ = this.searchQuerySubject.asObservable();

  updateQuery(query: string) {
    this.searchQuerySubject.next(query);
  }
}
```

### C. Global State Management Libraries (NgRx / NgRx SignalStore)

For complex enterprise applications with heavy client-side state, caching, and strict undo/redo or audit requirements:

- **NgRx Store:** Redux pattern (Actions, Reducers, Effects, Selectors).
- **NgRx SignalStore:** Modern, highly composable, signal-native store for enterprise feature modules.

---

## 4. Content Projection (`<ng-content>`)

When a parent needs to pass flexible DOM layouts or template fragments into a child component (e.g. modals, card wrappers, accordions):

```html
<div class="card-container">
  <div class="card-header">
    <ng-content select="[card-title]"></ng-content>
  </div>
  <div class="card-body">
    <ng-content></ng-content>
  </div>
</div>
```

---

## 5. ViewChild / ContentChild & Component Injection

When a parent component needs direct access to a child component's internal properties or public methods:

```typescript
@Component({ ... })
export class ParentComponent {
  private chartChild = viewChild.required(AnalyticsChartComponent);

  refreshChart() {
    this.chartChild().reRender();
  }
}
```

---

## 6. Route-Based Data Passing (Navigation)

When navigating between different pages/routes, data can be transferred via the Router:

1. Path / query parameters: `[routerLink]="['/user', userId()]` or `?tab=security`
2. Router state data (Navigation Extras):

```typescript
this.router.navigate(['/checkout'], { state: { orderData: this.currentOrder } });
```

3. Component input binding (`withComponentInputBinding()`):
   - Automatically maps query params, path params, and route `data` directly into component `input()` signals.

---

> *"In a large Angular application, we select the data-sharing mechanism based on component proximity and state complexity:*
> 1. *For direct parent-child relationships, we use Signal inputs (`input()`) for data flow down and `output()` emitters for events flowing up.*
> 2. *For shared UI layouts, we use Content Projection (`<ng-content>`).* 
> 3. *For cross-component or global state, we build Injectable Services using Signals (or NgRx SignalStore for complex state) to prevent prop-drilling.*
> 4. *For navigation-driven state, we leverage Router Component Input Bindings to pass route parameters directly into component inputs.*"

Q. What are route guards? How will you implement Page Not Found route? How to set query parameters?

**Route Guards** in Angular are interfaces/functions used to control access to routes based on programmatic conditions (e.g. authentication, permissions, form unsaved changes state, or data prefetching). They act as middleware for the Angular Router, deciding whether a navigation request should proceed, be cancelled, or be redirected.

In modern Angular (v14+-v21), **functional route guards** using `inject()` have replaced legacy class-based guards (`CanActivate` interface), providing a lighter, functional API.

### The 5 Types of Route Guards

1. `canActivate`: Determines if a user can visit a route.
2. `canActivateChild`: Determines if a user can access child routes of a parent feature.
3. `canDeactivate`: Prevents users from accidentally navigating away from a route with unsaved changes.
4. `canMatch` (replaced `canLoad`): Controls whether a route/lazy-loaded bundle can even be matched.
5. `resolve`: Prefetches required API data before the route component is rendered.

### Functional Guard Implementation Example

```typescript
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};
```

---

## How will you implement a "Page Not Found" (404) route?

A Page Not Found route catches any URL that does not match any defined path in your routing configuration.

### Implementation Strategy

1. Create a `NotFoundComponent`:

```typescript
@Component({
  selector: 'app-not-found',
  standalone: true,
  template: `
    <div class="error-container">
      <h1>404 - Page Not Found</h1>
      <p>The page you are looking for does not exist.</p>
      <a routerLink="/">Return Home</a>
    </div>
  `
})
export class NotFoundComponent {}
```

2. Configure the wildcard (`**`) route and place it last:

```typescript
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'dashboard', loadComponent: () => import('./dashboard/dashboard.component').then(m => m.DashboardComponent) },
  { path: '**', component: NotFoundComponent }
];
```

> **Lead Architect Note:** In SSR applications using Angular Universal, ensure your backend sets an actual HTTP `404` status code for the wildcard route.

---

## How to set query parameters?

Query parameters appear after a `?` in the URL (e.g. `/products?category=electronics&page=2`). In Angular, you can set query parameters declaratively or programmatically.

### Method 1: Template `queryParams`

```html
<a [routerLink]="['/products']"
   [queryParams]="{ category: 'shoes', page: 1 }">
  Filter Shoes
</a>
```

### Method 2: Programmatic Navigation

```typescript
@Component({ ... })
export class ProductListComponent {
  private router = inject(Router);

  applyFilter(categoryName: string) {
    this.router.navigate(['/products'], {
      queryParams: { category: categoryName, page: 1 },
      queryParamsHandling: 'merge'
    });
  }
}
```

### Reading Query Parameters

- Modern Signal Approach (Angular 16+): enable `withComponentInputBinding()` and declare query params as `input()` signals.

```typescript
category = input<string>();
```

- RxJS Approach (`ActivatedRoute`):

```typescript
export class ProductListComponent {
  private route = inject(ActivatedRoute);

  readonly category$ = this.route.queryParamMap.pipe(
    map(params => params.get('category') ?? 'all')
  );
}
```

Q. Here are the two ways to remove duplicate elements from an array in JavaScript / TypeScript.

---

## Method 1: Using Built-in Methods (`Set` or `filter`)

### Option A: `Set`

```typescript
const arr = [1, 2, 2, 3, 4, 4, 5];
const uniqueArr = [...new Set(arr)];

console.log(uniqueArr);
// Output: [1, 2, 3, 4, 5]
```

### Option B: `Array.prototype.filter()`

```typescript
const arr = [1, 2, 2, 3, 4, 4, 5];
const uniqueArr = arr.filter((item, index) => arr.indexOf(item) === index);

console.log(uniqueArr);
// Output: [1, 2, 3, 4, 5]
```

---

## Method 2: Using Traditional `for` Loop

```typescript
const arr = [1, 2, 2, 3, 4, 4, 5];
const uniqueArr: number[] = [];

for (let i = 0; i < arr.length; i++) {
  if (!uniqueArr.includes(arr[i])) {
    uniqueArr.push(arr[i]);
  }
}

console.log(uniqueArr);
// Output: [1, 2, 3, 4, 5]
```

Q. What is the role of `angular.json`?

In Angular, `angular.json` is the workspace configuration file. It acts as the central control panel for the entire project, defining how the Angular CLI builds, serves, tests, and deploys your application(s).

---

## 1. Primary Roles of `angular.json`

1. Project & multi-app architecture: defines all projects within a workspace.
2. Build configuration (`build` target): entry points, tsconfig, assets, styles, scripts.
3. Environment-specific overrides (`configurations`): build profiles for production, staging, development.
4. Tooling settings (`serve`, `test`, `extract-i18n`): local dev server, test runner, i18n extraction.

---

## 2. Key Sections of `angular.json`

```json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "my-angular-app": {
      "projectType": "application",
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": { ... },
        "serve": { ... },
        "test": { ... }
      }
    }
  }
}
```

### Critical Blocks inside `architect`

- `build`: tells Esbuild/Webpack how to compile the app.
- `index`: HTML template root.
- `main`: entry point TypeScript file.
- `assets`: static assets to copy.
- `styles`: global CSS/SCSS stylesheets.
- `scripts`: external JS files to include globally.

### `configurations` inside `build`

- `fileReplacements`: swaps environment files during production builds.
- `budgets`: warning/error thresholds for bundle sizes.
- `optimization`: enables tree-shaking, dead code elimination, and minification.

### `serve`

- Configures local development server, default port, SSL, and API proxy files.

---

Q. What is JIT and AOT? Difference between them?

In Angular, JIT (Just-In-Time) and AOT (Ahead-Of-Time) are compilation modes used to convert Angular templates and TypeScript into executable JavaScript.

---

## 1. High-Level Comparison Table

| Feature | Just-In-Time (JIT) | Ahead-Of-Time (AOT) |
| --- | --- | --- |
| When compilation happens | Inside the user's browser at runtime | On the build server during `ng build` at build time |
| Bundle size | Larger (includes Angular compiler bundle) | Smaller (compiler stripped out) |
| Initial load speed | Slower | Faster |
| Security & auditing | Less secure | More secure |
| Template error detection | Runtime | Build time |
| Default usage | Legacy local development | Default for modern Angular |

---

> *"JIT compiles Angular templates dynamically inside the user's browser at runtime, requiring us to ship the Angular compiler in the client bundle. AOT compiles templates into lightweight executable JavaScript ahead of time during the build process. Modern Angular uses AOT by default because it drastically reduces bundle sizes, speeds up initial page load, enhances security, and catches template binding errors at build time rather than at runtime."*

Q. How can a singleton service be shared only among a few components?

To restrict a service so that a single shared instance is shared only among a specific group of components, use Angular's hierarchical dependency injection system.

---

## 1. Provider at the Parent Component Level

If the components sharing the service exist within the same DOM subtree, register the service in the parent component's `providers` array.

### How it works

- Angular creates one single instance when the parent component initializes.
- The parent and all child/grandchild components receive the same shared instance.
- Components outside the tree cannot access or re-instantiate this service instance.

```typescript
@Injectable() // Do not set providedIn: 'root'
export class FeatureWidgetService {
  readonly activeTab = signal<string>('summary');
}

@Component({
  selector: 'app-widget-container',
  standalone: true,
  providers: [FeatureWidgetService],
  template: `
    <app-widget-header />
    <app-widget-body />
  `
})
export class WidgetContainerComponent {}

@Component({
  selector: 'app-widget-header',
  standalone: true,
  template: `<button (click)="service.activeTab.set('details')">Switch Tab</button>`
})
export class WidgetHeaderComponent {
  protected service = inject(FeatureWidgetService);
}

@Component({
  selector: 'app-widget-body',
  standalone: true,
  template: `<p>Active Tab: {{ service.activeTab() }}</p>`
})
export class WidgetBodyComponent {
  protected service = inject(FeatureWidgetService);
}
```

---

## 2. Using `@SkipSelf()` / `@Host()` Guards

If child components might accidentally re-provide the service locally, enforce consuming the existing parent instance:

```typescript
@Component({
  selector: 'app-widget-body',
  standalone: true,
  template: `...`
})
export class WidgetBodyComponent {
  private service = inject(FeatureWidgetService, { skipSelf: true });
}
```

---

## 3. Provider at the Route Level (Route-Scoped Singleton)

If components belong to a specific feature route or nested child routes, register the provider inside the route definition.

```typescript
import { Routes } from '@angular/router';
import { FeatureWidgetService } from './feature-widget.service';

export const FEATURE_ROUTES: Routes = [
  {
    path: 'feature',
    providers: [FeatureWidgetService],
    children: [
      { path: 'overview', component: FeatureOverviewComponent },
      { path: 'details', component: FeatureDetailsComponent }
    ]
  }
];
```

---

## Key Lead/Architect Summary

| Approach | Scope | Lifecycle | Best Used For |
| --- | --- | --- | --- |
| Parent component `providers` | Parent + all child components | Destroyed when parent component is destroyed | Compound UI widgets |
| Route `providers` | All components active on the route branch | Destroyed when user navigates away from the route | Shared state across feature pages/tabs |

> *"To share a singleton service exclusively among a subset of components, remove `providedIn: 'root'` and register the service in the `providers` array of the common ancestor--either at the parent component level or the route level. Through Angular's hierarchical DI, Angular creates a single instance at that node level, sharing it down to all nested components while isolating it from the rest of the application."*

Q. Difference between `@ViewChild` and `@ContentChild`?

- Use `@ViewChild` for elements written directly in the component's template and access them inside `ngAfterViewInit()`.
- Use `@ContentChild` for elements passed in from outside via `<ng-content>` and access them inside `ngAfterContentInit()`.

Q. What is Change Detection?

Change Detection is Angular's mechanism for synchronizing internal component state with the DOM. Historically, Angular relied on Zone.js to patch asynchronous APIs and perform top-down tree checks whenever an async operation completed.

In modern Angular (v18+), Zone.js is dropped. CD is triggered explicitly by:
- Signal writes
- Event bindings in templates (`(click)`, `(input)`)
- AsyncPipe emissions
- `markForCheck()` / `detectChanges()` called manually
- `afterNextRender()` / `afterRender()` hooks

Q. What are the ways Change Detection gets triggered in Angular?

In Angular, CD execution depends on whether the app is running Zoned or Zoneless:

- Zone.js Mode (Classic): Zone.js patches asynchronous browser APIs. When an async operation completes, Zone.js notifies Angular to run a top-down `ApplicationRef.tick()` across the entire component tree.
- Zoneless Mode (Modern - v18+): Without Zone.js, CD is scheduler-driven and explicit. It is triggered by:
  - Signal writes
  - Template event handlers
  - AsyncPipe emissions
  - explicit `ChangeDetectorRef` calls
  - `afterNextRender()` / `afterRender()` hooks

Key takeaway: In both modes, `ApplicationRef.tick()` performs the global pass, but Zoneless triggers it only when explicit reactive state changes occur.

Q. OnPush breaks my component data updates, but the view doesn't update. Why?

This is almost always caused by in-place object mutation.

Under `ChangeDetectionStrategy.OnPush`, Angular only runs change detection on a component if:
- An `@Input()` reference changes.
- An event originates from the component or one of its children.
- A Signal read in the template updates.
- `markForCheck()` is explicitly called.

If you mutate an array using `this.items.push(newItem)`, the array reference remains identical in memory. OnPush sees no input reference change and skips checking the view.

?? Fixes:
- Immutability: `this.items = [...this.items, newItem]`
- Use Signals: update a signal which automatically triggers view update.
- `ChangeDetectorRef.markForCheck()`: call manually after mutation.

Q. How do signals interact with OnPush?

Signals and OnPush complement each other:
- Reading a signal inside an OnPush template creates an implicit dependency.
- When the signal value changes, it automatically marks the component dirty for the next CD cycle.
- You get fine-grained reactive updates without needing immutable reference reassignment on every boundary.

Q. How do you avoid function calls in templates?

Direct function calls in templates (e.g. `{{ calculateTotal(items) }}`) execute on every change detection pass and can cause performance lag.

Avoid them using:
- `computed()` signals
- Pure pipes
- Pre-calculated component properties

Q. When would you use `ChangeDetectorRef.detectChanges()` vs `markForCheck()`?

| Feature | `markForCheck()` | `detectChanges()` |
| --- | --- | --- |
| Execution | Asynchronous / scheduled | Synchronous / immediate |
| Scope | Marks component and ancestors dirty | Immediately runs CD on current component and children |
| Use Case | Async updates, RxJS subscriptions, manual event bindings | Edge cases: imperative third-party DOM libraries |

Warning: overusing `detectChanges()` bypasses Angular's scheduler and can defeat the performance gains of OnPush.

Q (Trap). You have a `setInterval` updating a clock. Under OnPush, the view doesn't update. How do you fix it?

Since `setInterval` callbacks don't change `@Input()` references, OnPush skips checking the view.

Solutions:
1. Convert to a Signal:
   `clock = signal(new Date());`
   `this.clock.set(new Date());`
2. Inject `ChangeDetectorRef` and call `this.cdr.markForCheck()`.
3. Run outside Angular:

```typescript
this.ngZone.runOutsideAngular(() => {
  setInterval(() => {
    this.ngZone.run(() => this.clock.set(new Date()));
  }, 1000);
});
```

Q. How do you diagnose and fix a slow list render?

- Use Angular DevTools Profiler to measure CD duration and frame rate per row.
- Verify list tracking uses a stable key.
- Enforce OnPush on row child components.
- For 10k+ items, use virtual scroll (`cdk-virtual-scroll-viewport`).

Q. What is the cost difference between `AsyncPipe` and `selectSignal`?

- `AsyncPipe`: subscribes/unsubscribes to an Observable and calls `markForCheck()` on emission.
- `selectSignal` / `toSignal`: converts or reads data as a signal with no subscription lifecycle management.

Conclusion: under Zoneless Angular, `selectSignal` / Signals are higher performance.

Q. Zoneless Angular -- what actually changes?

- Setup: enabled via `provideExperimentalZonelessChangeDetection()` (v18+). Zone.js is removed from polyfills.
- What breaks: untracked async mutations no longer trigger UI refreshes automatically.
- What changes: all UI state updates must happen through Signals, template events, `AsyncPipe`, or `markForCheck()`.
- Benefits:
  - Smaller bundle size (~100 KB reduction)
  - Faster targeted change detection
  - Faster SSR and smoother hydration

Q. How do you measure whether OnPush is worth it on a component?

- Profile first with Angular DevTools.
- If a component's inputs rarely change while its parent updates frequently, OnPush saves render time.
- If a component already checks rarely, OnPush may have negligible difference.

Q. "The bundle size is 4 MB. Where do you start?"

- Analyze build stats with `ng build --stats-json`.
- Identify culprits:
  - Heavy `moment.js`: replace with `date-fns` or `Intl`.
  - Full lodash imports: use function-level imports.
  - Full UI library imports: import only needed modules.
- Verify lazy loading and ensure lazy routes/components are not imported directly in `app.config.ts` or root modules.

Q. Difference between Promise and Observable

- Promises are eager, single-value, and non-cancellable.
- Observables are lazy, multi-value, and cancellable.
- In modern Angular, Observables are used for complex async operations, bridged to Signals via `toSignal()`.

Q. How to add dynamic fields in Reactive Forms?

- Use `FormArray` for variable fields.
- Use `addControl()` / `removeControl()` on `FormGroup` for conditional fields.
- Enforce strictly typed form definitions and schema-driven factories for dynamic forms.

Q. In Angular routing: When navigating from Com1 ? Com2, how can we prevent the Back button from returning to Com1?

- Use `replaceUrl: true` with `navigate()` to replace history state.
- This prevents the browser Back button from returning to the login form or previous step.

Q. Demonstrate role-based routing (pseudo code)

```typescript
export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  {
    path: 'admin-dashboard',
    component: AdminDashboardComponent,
    canActivate: [roleGuard],
    data: { expectedRoles: ['ADMIN'] }
  },
  {
    path: 'reports',
    component: ReportsComponent,
    canActivate: [roleGuard],
    data: { expectedRoles: ['ADMIN', 'MANAGER'] }
  },
  { path: 'unauthorized', component: UnauthorizedComponent }
];

export const roleGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (!authService.isAuthenticated()) {
    return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
  }

  const expectedRoles = route.data['expectedRoles'] as string[];
  const userRoles = authService.getUserRoles();
  const hasPermission = expectedRoles.some(role => userRoles.includes(role));

  return hasPermission
    ? true
    : router.createUrlTree(['/unauthorized']);
};

@Injectable({ providedIn: 'root' })
export class AuthService {
  private userState = signal<{ name: string; roles: string[] } | null>(null);

  isAuthenticated(): boolean {
    return !!this.userState();
  }

  getUserRoles(): string[] {
    return this.userState()?.roles ?? [];
  }
}
```

Q. How does Virtual Scroll improve performance?

Virtual Scroll improves performance by rendering only the subset of items currently visible in the DOM viewport plus a small buffer, instead of instantiating thousands of DOM nodes for a large dataset.

Q. What is Deferred Loading?

Introduced in Angular 17, Deferred Loading (`@defer`) is a template-level block that enables fine-grained code splitting and lazy loading of UI components, directives, and pipes directly inside the template. Unlike route-level lazy loading, `@defer` lets you defer heavy UI elements until triggers such as viewport visibility, user hover, or idle time.

Q. What are Dumb and Smart Components?

The Smart vs. Dumb component pattern (Container vs. Presentational) separates concerns:
- Smart components manage state, business logic, and async calls.
- Dumb components are pure, stateless UI building blocks that accept data via inputs and emit events via outputs.

