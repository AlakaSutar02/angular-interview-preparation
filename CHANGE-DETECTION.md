Change detection (CD) is Angular's mechanism for keeping the DOM in sync with component state. 

The mental model: what triggers CD

## Under classic Zone.js:

Zone patches every async browser API (setTimeout, Promise, addEventListener, XHR/fetch, RAF). When any patched callback fires, Zone tells Angular "something happened", Angular runs CD from the root down. 
Every component in the tree gets checked unless it opts out.

That's the default, and it's expensive. A setInterval running every 100 ms triggers a full-app CD ten times a second whether anything visible changed or not.

## Under zoneless (Angular 18+, stable in 19):

Zone.js is dropped. CD is triggered explicitly by:

1. Signal writes (the framework knows which views depend on which signals)

2. Event bindings in templates ((click), (input))

3. AsyncPipe emissions

4. markForCheck()/ detectChanges() called manually

5. afterNextRender / afterRender hooks

Nothing else. setTimeout doesn't trigger CD unless you write a signal or call markForCheck inside it.

This is why signals matter for performance-they're the primary trigger under zoneless.

### ChangeDetectionStrategy.Default-Angular checks this component whenever CD runs anywhere in the tree.
### ChangeDetection Strategy. OnPush-Angular checks this component only when:

1. An @Input reference changes (--- inequality)

2. A DOM event fires from inside the component's template

3. An AsyncPipe in the template emits

4. A signal read in the template changes

5. changeDetectorRef.markForCheck() is called explicitly

That's it. If none of those happen, the component is skipped even if the parent runs CD.

Practical consequence: OnPush + immutable data + signals is the fast path. Mutating an object in place (this.items.push(x)) with OnPush → template does not update. That's the classic gotcha.

---

## Q1. What are the ways Change Detection (CD) gets triggered in Angular?

### 🎙️ Answer

In Angular, CD execution depends on whether the app is running **Zoned** or **Zoneless**:

* **Zone.js Mode (Classic):** Zone.js patches asynchronous browser APIs (`setTimeout`, `Promise`, HTTP events, DOM events). When an async operation completes, Zone.js notifies Angular to run a top-down `ApplicationRef.tick()` across the entire component tree.
* **Zoneless Mode (Modern - v18+):** Without Zone.js, CD is scheduler-driven and explicit. It is triggered by:
1. **Signal Writes** (e.g., updating a `signal()` or `linkedSignal()`).
2. **Template Event Handlers** (e.g., `(click)`).
3. **`AsyncPipe` Emissions**.
4. **Explicit Calls** to `ChangeDetectorRef.markForCheck()` or `detectChanges()`.
5. **`afterNextRender` / `afterRender` Hooks**.



> **Key Takeaway:** In both modes, `ApplicationRef.tick()` performs the global pass, but Zoneless triggers it only when explicit reactive state changes occur.

---

## Q2. OnPush breaks my component-data updates, but the view doesn't update. Why?

### 🎙️ Answer

This is almost always caused by **In-Place Object Mutation**.

Under `ChangeDetectionStrategy.OnPush`, Angular only runs change detection on a component if:

1. An `@Input()` reference changes (**Object identity check / `Object.is**`).
2. An event originates from the component or one of its children.
3. A Signal read in the template updates.
4. `markForCheck()` is explicitly called.

If you mutate an array using `this.items.push(newItem)`, the array reference remains identical in memory. `OnPush` sees no input reference change and skips checking the view.

### 🛠️ Fixes (In order of preference):

1. **Immutability (New Reference):** `this.items = [...this.items, newItem];`
2. **Use Signals:** Replace the property with a `signal()`—updating a signal automatically notifies the `OnPush` scheduler.
3. **`ChangeDetectorRef.markForCheck()`:** Call `this.cdr.markForCheck()` manually after mutating (legacy/fallback approach).

---

## Q3. How do signals interact with OnPush?

### 🎙️ Answer

Signals and `OnPush` complement each other perfectly:

* Reading a signal inside an `OnPush` component's template creates an implicit dependency.
* When the signal value changes, it **automatically marks the component (and its ancestors) as dirty for the next change detection cycle**.
* You get the benefit of fine-grained reactive updates **without needing immutable reference reassignment** on every boundary.

---

## Q4. How do you avoid function calls in templates?

### 🎙️ Answer

Direct function calls in templates (e.g., `{{ calculateTotal(items) }}`) execute on **every single change detection pass**, causing severe performance lag.

We avoid them using three options (ordered by preference):

1. **`computed()` Signals (Modern):** A `computed(() => ...)` signal automatically memoizes its value and only recalculates when its dependent signals change.
2. **Pure Pipes (Classic):** Angular pure pipes only re-evaluate when their input arguments change (via reference comparison).
3. **Pre-calculated Component Properties:** Compute the value once during initialization or state update and store it in a plain property.

---

## Q5. When would you use `ChangeDetectorRef.detectChanges()` vs `markForCheck()`?

### 🎙️ Answer

| Feature | `markForCheck()` | `detectChanges()` |
| --- | --- | --- |
| **Execution** | **Asynchronous / Scheduled** | **Synchronous / Immediate** |
| **Scope** | Marks component & ancestors dirty for the *next* global CD pass. | Immediately runs CD on the current component and its children. |
| **Use Case** | **95% of use cases:** Async updates, RxJS subscriptions, manual event bindings. | **Edge cases only:** Integrating imperative third-party DOM libraries where immediate layout calculation is required. |

> **Warning:** Overusing `detectChanges()` bypasses Angular's scheduler, causes redundant checks, and defeats the performance gains of `OnPush`.

---

## Q6 (Trap). You have a `setInterval` updating a clock. Under `OnPush`, the view doesn't update. How do you fix it?

### 🎙️ Answer

Since `setInterval` callbacks don't change `@Input()` references, `OnPush` skips checking the view.

### 🛠️ Solutions (Ranked Best to Worst):

1. **Convert to a Signal (Best practice):**
```typescript
clock = signal(new Date());
// Inside interval:
this.clock.set(new Date()); // Signal write automatically triggers view update under OnPush

```


2. **Inject `ChangeDetectorRef`:** Call `this.cdr.markForCheck()` inside the interval callback.
3. **Run Outside Angular (Optimization for high-frequency timers):**
```typescript
this.ngZone.runOutsideAngular(() => {
  setInterval(() => {
    // Only enter Angular Zone when updating UI state
    this.ngZone.run(() => this.clock.set(new Date()));
  }, 1000);
});

```



---

## Q7. How do you diagnose and fix a slow list render?

### 🎙️ Answer

1. **Diagnosis:** Use Angular DevTools Profiler to measure the change detection duration and frame rate per row.
2. **Verify List Tracking:** Ensure the `@for` block uses a stable tracking key (e.g., `@for (item of items; track item.id)`). Avoid tracking by array index or random values.
3. **Enforce `OnPush` on Children:** Ensure row components use `OnPush` with stable immutable inputs.
4. **Virtualization (For 10k+ items):** Use `@angular/cdk/scrolling` (`cdk-virtual-scroll-viewport`) to render only the DOM elements currently visible in the viewport.

---

## Q8. What is the cost difference between `AsyncPipe` and `selectSignal`?

### 🎙️ Answer

* **`AsyncPipe`:** Subscribes and unsubscribes to an RxJS Observable automatically and calls `markForCheck()` on emission. It carries memory and execution overhead from managing RxJS subscription lifecycles.
* **`selectSignal` / `toSignal`:** Converts or reads data as a Signal. It has **no subscription lifecycle management**, operates via direct signal reads, and integrates directly with Angular’s fine-grained change detection scheduler.

> **Conclusion:** Under **Zoneless Angular**, `selectSignal` / Signals is the recommended, higher-performance path.

---

## Q9. Zoneless Angular — what actually changes?

### 🎙️ Answer

* **Setup:** Enabled via `provideExperimentalZonelessChangeDetection()` (v18+). `Zone.js` is removed entirely from `angular.json` polyfills.
* **What Breaks:** Un-tracked async mutations (e.g., a raw `setTimeout` updating a plain class field) will no longer trigger UI refreshes automatically.
* **What Changes:** All UI state updates must happen through **Signals, template events, `AsyncPipe`, or `markForCheck()**`. Third-party libraries relying on Zone-patched APIs must be audited.
* **Benefits:**
* Smaller bundle size (**~100 KB reduction** without Zone.js).
* Faster, targeted change detection.
* Faster SSR (Server-Side Rendering) and smoother hydration.



---

## Q10. How do you measure whether `OnPush` is worth it on a component?

### 🎙️ Answer

Don't blanket-apply `OnPush` without profiling.

1. **Profile First:** Use **Angular DevTools Profiler** to record CD passes during application interactions.
2. **Evaluate Impact:**
* If a component sits inside a parent that updates frequently, but the component's own inputs rarely change, `OnPush` saves significant render time.
* If a component already checks rarely or has no child tree, `OnPush` produces negligible difference.



---

## Q11. "The bundle size is 4 MB. Where do you start?"

### 🎙️ Answer

1. **Analyze Build Stats:**
Run `ng build --stats-json` and inspect the bundle breakdown using **`source-map-explorer`** or `webpack-bundle-analyzer`.
2. **Identify Common Culprits:**
* **Heavy Moment.js:** Replace with `date-fns` or native JavaScript `Intl` APIs.
* **Full Lodash Imports:** Change `import _ from 'lodash'` to function-level imports `import debounce from 'lodash/debounce'`.
* **Full UI Library Imports:** Ensure component-level imports (e.g., `@angular/material`) rather than importing entire modules.


3. **Verify Lazy Loading:**
* Inspect route configurations to ensure feature modules/components use `loadComponent` or `loadChildren`.
* **Common Trap:** Ensure lazy-loaded components aren't accidentally imported directly in `app.config.ts` or root modules, which pulls them back into the main bundle.