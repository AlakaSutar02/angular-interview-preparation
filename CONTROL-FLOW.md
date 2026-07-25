---

# **Angular Built-in Control Flow Prep Guide**

## **Why Built-in Control Flow? (Old vs. New)**

### **Old Way (`*ngIf`, `*ngFor`, `[ngSwitch]`)**

* **Performance overhead:** Depended on runtime directive lookup and instantiation.
* **Fragile type narrowing:** `*ngIf="user as u"` worked, but nested or composed logic broke easily.
* **Missing semantics:** No built-in way to handle loading, empty, or deferred states—you had to hand-roll them.
* **No directive composition:** Combining directives on a single element (e.g., `*ngIf` and `*ngFor`) was illegal.

### **New Way (`@if`, `@for`, `@switch`, `@defer`)**

* **Compiler-level optimization:** The template compiler emits branching logic directly with no directive overhead.
* **Superior type narrowing:** `@if (user) { {{ user.name }} }` automatically narrows `user` from `User | null` to `User` without needing `as`.
* **Built-in features:** First-class `@else if`, `@else`, `@empty` in `@for`, and mandatory `track`.
* **Performance & Bundle size:** Benchmarks show up to **~90% faster** rendering for `@for` vs `*ngFor` on large lists. Eliminates the need to import `CommonModule` for control flow.

---

## **Syntax Reference & Key Rules**

### **1. `@if` / `@else if` / `@else**`

```html
@if (customer(); as c) {
<app-customer-card [customer]="c" />
} @else if (isLoading()) {
<ui-spinner />
} @else {
<p>No customer selected.</p>
}
```

- **Scope Isolation:** Each block branch forms its own scope—variables declared in `@if` aren't accessible inside `@else`.
- **Optional `as` Binding:** The `as` binding is optional but useful when aliasing or narrowing continuous signal reads.

---

### **2. `@for` with Mandatory `track` & `@empty**`

```html
@for (payment of payments(); track payment.id; let i = $index, isFirst = $first)
{
<app-payment-row [payment]="payment" [isFirst]="isFirst" />
} @empty {
<p>No payments yet.</p>
}
```

#### **Key Rules & Changes:**

- **`track` is mandatory:** It takes an **expression** directly rather than a helper function reference (`trackById`).
- **Old:** `<div *ngFor="let p of payments; trackBy: trackById">`
- **New:** `@for (p of payments; track p.id)`

- **Composite Keys:** If an ID requires multiple fields, inline it: `track p.customerId + ':' + p.date`.
- **`track $index`:** Valid syntax, but consider it a code smell (only use for static or append-only lists where item identities never shift).
- **Contextual Variables:** `$index`, `$first`, `$last`, `$even`, `$odd`, `$count` remain available.

---

### **3. `@switch**`

```html
@switch (status()) { @case ('pending') {
<ui-badge tone="warn">Pending</ui-badge>
} @case ('paid') {
<ui-badge tone="success">Paid</ui-badge>
} @case ('failed') {
<ui-badge tone="danger">Failed</ui-badge>
} @default {
<ui-badge>Unknown</ui-badge>
} }
```

- **Strict Equality:** Uses strict equality (`===`), unlike the old `[ngSwitch]` which allowed loose equality checks (`==`).
- **No Fall-through:** Cases do not fall through to adjacent blocks. `@default` is optional.

---

### **4. `@defer` (Block-Level Lazy Loading)**

Pushes code-splitting down to individual component sub-trees rather than relying strictly on route boundaries.

```html
@defer (on viewport; prefetch on idle) {
<app-heavy-chart [data]="chartData()" />
} @loading (minimum 200ms) {
<ui-skeleton />
} @error {
<p>Chart failed to load.</p>
} @placeholder (minimum 500ms) {
<div class="chart-placeholder"></div>
}
```

- **Triggers:** Supports interaction, viewport scrolling, timers, or custom boolean conditions to stream JS bundles on demand.
- **Sub-blocks:** Accepts `@placeholder`, `@loading`, and `@error` blocks with optional timing parameters (`minimum` / `after`) to prevent layout flickering.

Here is how a **human Senior / Staff Developer (9+ years experience)** actually answers these questions in an interview—speaking naturally, using real-world scenarios, and walking through code examples.

---

### **Q1. "Why did Angular introduce built-in control flow when structural directives worked?"**

> **How to answer:**
> "The structural directives like `*ngIf` and `*ngFor` were great, but under the hood, they were runtime directive classes. Every time you used `*ngFor`, Angular had to instantiate a directive, inject dependencies, and manage a `ViewContainerRef`.
> With built-in control flow, branching moves straight into the **Angular Template Compiler**. It emits simple JavaScript branching code directly—no directive instances at runtime, which gives us a massive performance boost, especially on large lists. Plus, type narrowing just works inside the block without ugly `as` syntax, and we get features like `@defer` baked into the language."

**Example:**

```html
<!-- OLD: Directive class instantiation + fragile narrowing -->
<div *ngIf="user$ | async as user">
  <p>{{ user.name }}</p>
</div>

<!-- NEW: Compiler-native branch with automatic type narrowing -->
@if (user(); as u) {
<p>{{ u.name }}</p>
} @else {
<p>No user found</p>
}
```

---

### **Q2. "What is `track` and why is it required?"**

> **How to answer:**
> "`track` tells Angular how to uniquely identify items in a list so it knows which DOM nodes to reuse when data updates.
> Under `*ngFor`, tracking defaulted to object reference check (`===`). But in modern Angular apps where we use immutable state—like NgRx or Signals—refetching data returns brand-new object references even if the underlying data is identical. Without explicit tracking, Angular would destroy and recreate the whole DOM list, killing performance and wiping out active focus or input states. Making `track` mandatory forces us to define a unique key upfront."

**Example:**

```html
<!-- Forces Angular to track items by ID instead of object memory reference -->
@for (item of products(); track item.id) {
<app-product-card [product]="item" />
} @empty {
<p>No products available.</p>
}
```

---

### **Q3. "When would `track $index` be correct?"**

> **How to answer:**
> "Only when the list is purely **static, read-only, or append-only**, and the items don't have a unique primary key—like rendering a stream of real-time log entries or a fixed list of string labels.
> If the user can filter, reorder, delete, or insert items mid-list, using `$index` is an anti-pattern because array indices change position, causing Angular to bind old component state to new data."

**Example:**

```html
<!-- Good use case: Static array of strings with no unique IDs -->
@for (log of systemLogs(); track $index) {
<p class="log-line">{{ log }}</p>
}
```

---

### **Q4. "Explain `@defer` and when you'd reach for it."**

> **How to answer:**
> "`@defer` gives us declarative, block-level lazy loading directly in the template. Instead of splitting bundles only at the route level, we can split heavy component chunks anywhere on the page.
> I reach for `@defer` when a view has heavy third-party dependencies—like a charting library, Monaco editor, or a PDF viewer—that aren't needed for the initial page render. It keeps our main bundle small and speeds up initial load."

**Example:**

```html
<!-- Heavily delays loading the Chart JS chunk until the user scrolls it into view -->
@defer (on viewport; prefetch on idle) {
<app-heavy-analytics-chart [data]="chartData()" />
} @placeholder {
<div class="chart-skeleton">Loading chart area...</div>
} @loading (minimum 300ms) {
<ui-spinner />
}
```

---

### **Q5. "Difference between `on viewport` and `on interaction`?"**

> **How to answer:**
> "`on viewport` uses the browser's `IntersectionObserver` under the hood. It triggers the chunk download as soon as the placeholder element physically scrolls into view. That's great for below-the-fold content.
> `on interaction` waits until the user explicitly interacts with the placeholder element—like clicking or focusing it. That's ideal for things like tab contents, dropdowns, or popovers where you only want to load code if the user actually intends to use that UI."

**Example:**

```html
<!-- Triggers download ONLY when user clicks the placeholder button -->
@defer (on interaction(loadBtn)) {
<app-comments-section />
} @placeholder {
<button #loadBtn>Click to view comments</button>
}
```

---

### **Q6. "`@switch` vs the old `[ngSwitch]` — any behavior difference?"**

> **How to answer:**
> "Yes, `@switch` uses **strict equality (`===`)**, whereas `[ngSwitch]` used loose equality (`==`).
> If you have legacy code where a string ID like `'10'` was being compared against a number `10`, `[ngSwitch]` matched it, but `@switch` will evaluate to false. You have to make sure your types match strictly."

**Example:**

```html
<!-- Must strictly match string to string or number to number -->
@switch (userRole()) { @case ('ADMIN') { <app-admin-panel /> } @case ('USER') {
<app-user-panel /> } @default { <app-guest-panel /> } }
```

---

### **Q7 (Trap). "Why does my `@if` block re-render its whole subtree even for tiny changes?"**

> **How to answer:**
> "Because `@if` is a structural view boundary. Whenever its condition toggles truthiness or changes reference, Angular tears down the entire inner DOM view and rebuilds it from scratch.
> This usually happens when the condition calls a function or a `computed()` signal that returns a new object or array instance on every change detection pass. The fix is to make sure your condition returns a stable boolean primitive or a memoized value."

**Example:**

```typescript
// BAD: Returning a new object reference every pass triggers constant teardowns
condition = computed(() => ({ active: this.status() === "ACTIVE" }));

// GOOD: Return a simple boolean primitive
isStatusActive = computed(() => this.status() === "ACTIVE");
```

---

### **Q8. "Can you use control-flow syntax with `<ng-template>`?"**

> **How to answer:**
> "You don't need `<ng-template>` for standard structural branching anymore because `@if`, `@for`, and `@switch` create their own view boundaries natively. Old patterns like `<ng-template *ngIf>` are obsolete.
> However, `<ng-template>` is still crucial for dynamic template outlets (`ngTemplateOutlet`), content projection, and building reusable UI component templates."

**Example:**

```html
<!-- <ng-template> used for dynamic injection via ngTemplateOutlet -->
<ng-container *ngTemplateOutlet="customHeader ? customHeader : defaultHeader" />

<ng-template #defaultHeader>
  <h2>Default Title</h2>
</ng-template>
```

---

### **Q9. "Any downside to always using `@defer`?"**

> **How to answer:**
> "Yes, network fragmentation. Every `@defer` block creates a separate JavaScript bundle.
> If you over-use it on tiny components (under 30–50 KB), the overhead of making multiple HTTP requests, parsing small chunks, and tracking hydration state actually makes the app feel slower than if you had just bundled them together. You should reserve `@defer` for genuinely heavy dependencies or large subtrees."

---

### **Q10. "What does the compiler produce for `@for`?"**

> **How to answer:**
> "Instead of instantiating directive instances and managing `ViewContainerRef` containers for every iteration, the compiler emits a plain, high-performance imperative loop that manages an internal lookup map of `track` keys to DOM view references.
> When state updates, it diffs keys against the map:
>
> 1. **Existing key:** Reuses the existing DOM nodes and runs local updates.
> 2. **Removed key:** Destroys that view node.
> 3. **New key:** Instantiates a new DOM node."
