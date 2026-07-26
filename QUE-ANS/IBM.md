Here are lead-architect level interview responses, precise outputs, code solutions, and real-world system explanations for all 15 questions.

---

### **1) Event Loop with Execution Context and Call Stack**

> "When JavaScript code executes, the engine creates an **Execution Context** consisting of two phases: the **Memory Creation Phase** (hoisting variables/functions) and the **Execution Phase**.
>
> 1. **Call Stack:** A LIFO (Last-In, First-Out) stack that tracks function calls and their execution contexts. Synchronous code executes immediately here.
> 2. **Web APIs / Node Runtimes:** Asynchronous operations (HTTP calls, timers, DOM events) are offloaded to browser/Node threads.
> 3. **Task Queues:**
>
> - **Microtask Queue:** Holds high-priority callbacks like `Promise.then()`, `queueMicrotask()`, and `MutationObserver`.
> - **Macrotask Queue (Callback Queue):** Holds tasks like `setTimeout()`, `setInterval()`, and `requestAnimationFrame()`.
>
> 4. **Event Loop:** A continuous process that monitors the Call Stack. When the Call Stack is completely **empty**, the Event Loop first drains **ALL microtasks** sequentially. Only after the Microtask Queue is completely clear does it pick **ONE macrotask** from the Macrotask Queue and push its callback onto the Call Stack. It repeats this cycle indefinitely."

---

### **2) How JavaScript Handles Asynchronous Operations Despite Being Single-Threaded**

> "JavaScript's runtime engine (V8, SpiderMonkey) is single-threaded—it has only **one Call Stack** and executes one line of code at a time on the main thread.
> However, JavaScript runs inside a **hosting environment** (Browser or Node.js) that is **multi-threaded**. When an asynchronous API like `fetch()` or `setTimeout()` is invoked, the JS engine passes the task to browser Web APIs (written in C++). The background C++ worker threads handle network requests or timer countdowns without blocking the JS execution thread. When the task finishes, the C++ thread drops the callback into the Event Loop's queues. JavaScript itself remains single-threaded, but it delegates multi-threaded work to its runtime environment."

---

### **3) What is Closure? (With Practical Example)**

> "A **closure** is the combination of a function bundled together (enclosed) with references to its surrounding state (**lexical environment**). In JavaScript, every inner function retains access to variables declared in its outer scope, even after that outer function has finished executing and returned.
> **Architectural Use Cases:** Data privacy/encapsulation, factory functions, memoization, and partial application."

```typescript
function createCounter(initialValue: number = 0) {
  // Private variable encapsulated via lexical closure
  let count = initialValue;

  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
}

const counter = createCounter(10);
console.log(counter.increment()); // 11
console.log(counter.increment()); // 12
console.log(counter.getCount()); // 12
// count is inaccessible directly from outside: counter.count is undefined!
```

---

### **Outputs for Given Snippets**

#### **Snippet A:**

```javascript
let x = 11;
function find() {
  if (true) {
    let x = 12;
    console.log(x);
  }
  console.log(x);
}
find();
```

**Output:**

```text
12
11

```

_Reason:_ `let` is block-scoped. Inside the `if` block, `let x = 12` shadows outer `x`. Outside the `if` block (inside `find()`), it accesses the outer lexical scope `x = 11`.

---

#### **Snippet B:**

```javascript
async function test() {
  console.log(1);
  await Promise.resolve().then(() => {
    console.log("promise");
  });
  setTimeout(() => {
    console.log("timeout");
  }, 0);
  console.log(2);
}
test();
```

**Output:**

```text
1
promise
2
timeout

```

_Step-by-step Execution Order:_

1. `console.log(1)` runs synchronously $\rightarrow$ Prints **`1`**.
2. `Promise.resolve().then()` registers a callback in the **Microtask Queue**. `await` pauses execution of `test()`.
3. Microtask Queue executes $\rightarrow$ Prints **`promise`**.
4. The remaining body of `test()` resumes execution:

- `setTimeout(..., 0)` registers its callback in the **Macrotask Queue**.
- `console.log(2)` executes synchronously $\rightarrow$ Prints **`2`**.

5. Call Stack and Microtasks are now clear. Event Loop picks the Macrotask $\rightarrow$ Prints **`timeout`**.

---

### **4) Mask Email Solution**

#### **Requirement Rules:**

- Filter out `null` and `undefined`.
- For email usernames:
- Length 1: Keep letter (`a@gmail.com`).
- Length 2: Keep 1st letter + `*` (`a*@gmail.com`).
- Length > 2: Keep 1st & last letter, mask inside with `*` (`j******e@gmail.com`).

```typescript
function maskEmails(emails: (string | null | undefined)[]): string[] {
  return emails
    .filter(
      (email): email is string =>
        typeof email === "string" && email.includes("@"),
    )
    .map((email) => {
      const [username, domain] = email.split("@");
      const len = username.length;

      let maskedUser = "";
      if (len <= 1) {
        maskedUser = username;
      } else if (len === 2) {
        maskedUser = `${username[0]}*`;
      } else {
        maskedUser = `${username[0]}${"*".repeat(len - 2)}${username[len - 1]}`;
      }

      return `${maskedUser}@${domain}`;
    });
}

// Execution
const input = [
  "john.doe@gmail.com",
  "aem@gmail.com",
  undefined,
  "ab@gmail.com",
  "user123@yahoo.com",
  null,
];
console.log(maskEmails(input));
// Output: ['j******e@gmail.com', 'a*m@gmail.com', 'a*@gmail.com', 'u*****3@yahoo.com']
```

---

### **5) Change Detection & `OnPush` Use Case**

> "Change Detection is Angular’s process of traversing the component tree to check for updated state values and synchronizing them with the DOM.
>
> - **`Default` Strategy:** Angular checks every component in the entire component tree on every single DOM event, timer, or HTTP response.
> - **`OnPush` Strategy (`ChangeDetectionStrategy.OnPush`):** Tells Angular to **skip** checking a component subtree unless:
>
> 1. An `@Input()` reference changes (immutable change).
> 2. An event listener bound inside the component/template fires.
> 3. A Signal read in the template emits a change.
> 4. `ChangeDetectorRef.markForCheck()` is called manually.
>
> **Production Use Case:** In large data tables, dashboards with high event frequency, or real-time trading dashboards, using `OnPush` reduces unnecessary dirty checks from thousands per second down to single digits."

---

### **6) What is `Zone.js`?**

> "`Zone.js` is an execution context monkey-patching library. It overrides standard asynchronous browser APIs (`setTimeout`, `Promise`, `addEventListener`, `XHR`) with customized zone wrappers.
> When an async task completes anywhere in an Angular app, `Zone.js` detects it and notifies Angular (`ngZone.onUnstable` / `onMicrotaskEmpty`), automatically triggering a top-down Change Detection run.
> _Modern Angular Shift:_ Angular 18+ introduced official support for **Zoneless applications** (`provideZonelessChangeDetection()`), removing `Zone.js` overhead entirely by relying on **Signals** for fine-grained reactivity."

---

### **7) Debugging Dashboard Widgets & Memory Leak Strategy**

#### **Scenario A: 3 Out of 5 Widgets Failing to Load**

1. **Network Tab Inspection:** Check if API calls for the 3 failed widgets returned 4xx/5xx errors, timed out, or were blocked by CORS.
2. **Console & Boundary Interceptor:** Verify if unhandled JS runtime exceptions were thrown in widget components.
3. **HTTP Interceptor Isolation:** Check if individual widget request failures are silently breaking parent observables (`forkJoin` without error catching per stream).
4. **Auth / Permissions Token:** Verify if the user lacks authorization scopes for those specific endpoint datasets.

#### **Scenario B: Identifying and Fixing Memory Leaks**

1. **Detection:**

- Take a **Heap Snapshot** in Chrome DevTools under Memory tab.
- Interact with the application (e.g., navigate to a view and return 10 times).
- Take a second Heap Snapshot and run a **Comparison Profile**. Look for detached DOM elements or un-destroyed Component classes (`Detached HTMLDivElement`).

2. **Resolution Strategies:**

- **RxJS Subscriptions:** Unsubscribe properly using `takeUntilDestroyed()`, `takeUntil()`, or replace explicit subscriptions with the `async` pipe / Signals (`toSignal()`).
- **DOM Listeners:** Ensure raw `window.addEventListener()` or `setInterval()` references are cleared inside `ngOnDestroy`.
- **Global Event Busses:** Remove listeners on global BehaviorSubjects when components destroy.

---

### **8) Angular Application Performance Optimization Techniques**

> 1. **Code Splitting & Deferred Views:** Implement route-level lazy loading and template `@defer (on viewport)` for heavy components.
> 2. **Change Detection Modernization:** Enforce `ChangeDetectionStrategy.OnPush` or migrate to Zoneless architecture with Signals.
> 3. **DOM Rendering:** Use CDK Virtual Scrolling (`cdk-virtual-scroll-viewport`) for long lists and optimize `@for` loops with explicit `track` clauses.
> 4. **Asset Optimization:** Use `NgOptimizedImage` for lazy loading pictures with explicit aspect ratios to eliminate cumulative layout shifts (CLS).
> 5. **Build Tuning:** Enable AOT compilation, build-time tree shaking, and configure HTTP response caching via Interceptors.

---

### **9) Protecting Applications from XSS (Cross-Site Scripting)**

> 1. **Built-in Sanitization:** Angular treats all values as untrusted by default. It automatically sanitizes interpolated values in HTML templates (`{{ val }}`) and property bindings.
> 2. **Avoid Bypass Security APIs:** Do not call `DomSanitizer.bypassSecurityTrustHtml()` unless strictly verified and sanitized server-side.
> 3. **Avoid Raw DOM Manipulation:** Avoid using `ElementRef.nativeElement.innerHTML = userInput`. Use `Renderer2` or native bindings instead.
> 4. **Content Security Policy (CSP):** Enforce strict HTTP response headers:
>
> ```http
> Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.cdn.com; object-src 'none';
>
> ```
>
> 5. **HttpOnly & SameSite Cookies:** Store sensitive user tokens in `HttpOnly`, `Secure` cookies to render them inaccessible to client-side scripts.

---

### **10) Implementing Multi-Layer Encryption in Web Applications**

> "Multi-layer security covers data in transit, data at rest, and payload-level encryption:
>
> 1. **Transport Layer Security (TLS 1.3):** First layer protecting all data in transit over HTTPS.
> 2. **Payload-Level Asymmetric Encryption (Web Crypto API / RSA-OAEP):** Before sensitive payloads (e.g., credit card / SSN) leave the Angular app, encrypt them on the client side using a public key provided by the server:
>
> ```typescript
> const encryptedData = await window.crypto.subtle.encrypt(
>   { name: "RSA-OAEP" },
>   serverPublicKey,
>   new TextEncoder().encode(JSON.stringify(payload)),
> );
> ```
>
> 3. **Token Encapsulation (JWE - JSON Web Encryption):** Encrypt sensitive JSON claims into a compact encrypted token format rather than standard unencrypted JWT (JWS).
> 4. **Database-Level Encryption:** Server decrypts payload using private keys held in HSM (Hardware Security Module) before storing with AES-256 at rest."

---

### **11) Subject vs BehaviorSubject vs ReplaySubject vs AsyncSubject**

| Subject Type           | Initial Value Required? | Emits to New Subscribers                                               | Primary Use Case                                    |
| ---------------------- | ----------------------- | ---------------------------------------------------------------------- | --------------------------------------------------- |
| **`Subject`**          | No                      | Only future values emitted _after_ subscription.                       | Imperative event bus notifications.                 |
| **`BehaviorSubject`**  | **Yes**                 | The **last emitted value** immediately upon subscription.              | Reactive store state (e.g., `currentUser$`).        |
| **`ReplaySubject(N)`** | No                      | The last **$N$ emitted values**, regardless of subscription timing.    | Caching recent history or buffered logs.            |
| **`AsyncSubject`**     | No                      | **Only the single final value** upon stream completion (`complete()`). | Single execution calculations (like HTTP requests). |

---

### **12) RxJS Combination & Flattening Operators (Code & Network Behavior)**

```typescript
import { HttpClient } from "@angular/common/http";
import { inject, Injectable } from "@angular/core";
import { concatMap, exhaustMap, forkJoin, mergeMap, of, switchMap } from "rxjs";

@Injectable({ providedIn: "root" })
export class RxjsDemoService {
  private http = inject(HttpClient);

  // 1. forkJoin: Waits for ALL observables to complete, emits last values together
  // Network Tab: Sends API 1 & API 2 concurrently. Nothing processes until BOTH finish.
  loadDashboardData() {
    return forkJoin({
      user: this.http.get("/api/user"),
      stats: this.http.get("/api/stats"),
    });
  }

  // 2. switchMap: Cancels previous pending inner request when a new emission arrives
  // Network Tab: If user types "a", "ab", "abc", fast, pending HTTP calls for "a" & "ab" show "Canceled" in red.
  search(query$: Observable<string>) {
    return query$.pipe(switchMap((q) => this.http.get(`/api/search?q=${q}`)));
  }

  // 3. mergeMap: Executes inner observables concurrently without order guarantee
  // Network Tab: Fires multiple requests at once; completes in whatever order server responds.
  fetchItemsConcurrently(ids$: Observable<number[]>) {
    return ids$.pipe(
      mergeMap((ids) => ids.map((id) => this.http.get(`/api/item/${id}`))),
    );
  }

  // 4. concatMap: Executes inner observables sequentially in strict order
  // Network Tab: Queue behavior. Call #2 starts ONLY after Call #1 finishes completely.
  saveSequentially(items$: Observable<any[]>) {
    return items$.pipe(concatMap((item) => this.http.post("/api/save", item)));
  }

  // 5. exhaustMap: Ignores incoming outer emissions while current inner observable is running
  // Network Tab: Prevents duplicate submissions. Subsequent clicks ignored while request is pending.
  submitForm(buttonClick$: Observable<void>) {
    return buttonClick$.pipe(
      exhaustMap(() =>
        this.http.post("/api/submit", { timestamp: Date.now() }),
      ),
    );
  }
}
```

---

### **13) Why Signals alongside RxJS? Types, `.set()` vs `.update()**`

> **Why Signals when we have RxJS?**
>
> - **RxJS** is an asynchronous stream library ($O(N)$ event composition). It excels at orchestration, events, and API pipelines, but is heavy for static state binding.
> - **Signals** are fine-grained synchronous reactive primitives ($O(1)$ value tracking). They notify Angular **exactly which DOM node needs updating**, enabling high performance and Zoneless rendering without RxJS subscriber overhead.

#### **Signal Types:**

1. **Writable Signal:** `signal(initialVal)`
2. **Computed Signal:** `computed(() => val() * 2)`
3. **Linked Signal:** `linkedSignal({ source: A, computation: () => ... })`

#### **Difference Between `.set()` and `.update()**`

- **`.set(newValue)`:** Directly replaces the current signal value without reading previous state.

```typescript
const count = signal(0);
count.set(10); // Directly replaces with 10
```

- **`.update(fn)`:** Computes a new value based on the current existing state value.

```typescript
const count = signal(10);
count.update((current) => current + 1); // Uses existing 10 -> returns 11
```

---

### **14) Interceptors & API Mocking**

> "An **HTTP Interceptor** sits in the Angular `HttpClient` middleware pipeline to intercept and transform outgoing requests and incoming responses globally.
> **Can Interceptors be used for API Mocking?**
> **Yes.** You can build a mock interceptor that intercepts target endpoint requests, bypasses the network completely, and returns an RxJS `HttpResponse` containing mock JSON data with simulated network delay."

```typescript
import {
  HttpHandlerFn,
  HttpInterceptorFn,
  HttpRequest,
  HttpResponse,
} from "@angular/common/http";
import { delay, of } from "rxjs";

export const mockApiInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn,
) => {
  // Intercept target mock endpoint
  if (req.url.endsWith("/api/v1/user-profile")) {
    const mockUser = { id: 101, name: "Lead Architect", role: "ADMIN" };

    // Bypass backend and return fake HTTP Response
    return of(new HttpResponse({ status: 200, body: mockUser })).pipe(
      delay(500), // Simulate 500ms network latency
    );
  }

  // Pass unmatched requests to real server
  return next(req);
};
```

---

### **15) SCSS Functions, Mixins, and `@include` Demonstration**

```scss
// 1. SCSS Function: Calculates values and RETURNS a computed value
@function calculate-rem($pixelValue) {
  @return ($pixelValue / 16) * 1rem;
}

// 2. SCSS Mixin: Encapsulates reusable blocks of CSS declarations & arguments
@mixin flex-center($direction: row, $justify: center) {
  display: flex;
  flex-direction: $direction;
  justify-content: $justify;
  align-items: center;
}

// Responsive Breakpoint Mixin
@mixin mobile-view {
  @media (max-width: 768px) {
    @content; // Injects nested CSS rules passed from consumer
  }
}

// CONSUMPTION USING @include
.dashboard-card {
  // Using function
  padding: calculate-rem(32); // Converts 32px -> 2rem
  font-size: calculate-rem(16);

  // Using mixin with @include
  @include flex-center(column, space-between);

  // Using responsive mixin with content block
  @include mobile-view {
    padding: calculate-rem(16);
    width: 100%;
  }
}
```
