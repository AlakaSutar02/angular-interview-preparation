Here are clear, senior-level interview responses with code examples for each question:

---

### **1) Difference Between Block and Inline Elements**

| Feature        | Block Elements                                                     | Inline Elements                                                                                  |
| -------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| **Line Flow**  | Starts on a **new line** and stretches full width available.       | Fits within the **flow of text** without starting a new line.                                    |
| **Dimensions** | Respects `width`, `height`, `margin`, and `padding` (all 4 sides). | Ignores `width` & `height`. Respects horizontal margins/padding, but ignores top/bottom margins. |
| **Examples**   | `<div>`, `<p>`, `<h1>`-`<h6>`, `<section>`, `<ul>`                 | `<span>`, `<a>`, `<strong>`, `<em>`, `<img>`                                                     |

---

### **2) What are Semantic HTML Elements?**

> "Semantic HTML elements clearly describe their meaning to both the browser and the developer in human-readable terms. Instead of using non-semantic generic tags like `<div class="header">`, semantic elements like `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, and `<footer>` clearly define the structural role of the content.
> **Why they are important:**
>
> 1. **Accessibility (a11y):** Screen readers use semantic landmarks to help visually impaired users navigate pages easily.
> 2. **SEO:** Search engines prioritize structured semantic markup to index page hierarchy accurately.
> 3. **Maintainability:** Makes code far easier to read and maintain across teams."

---

### **3) What are the Different CSS Positions?**

| Position       | Behavior                                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **`static`**   | **Default value.** Follows normal page document flow. Properties like `top`, `left`, `z-index` have no effect.            |
| **`relative`** | Positioned relative to its **own normal position** in document flow without affecting surrounding elements.               |
| **`absolute`** | Removed from document flow and positioned relative to its **nearest positioned ancestor** (anything other than `static`). |
| **`fixed`**    | Removed from document flow and positioned relative to the **viewport/window**. Stays in place during scrolling.           |
| **`sticky`**   | Toggles between `relative` and `fixed` depending on scroll offset relative to its scroll container threshold.             |

---

### **4) Difference Between Relative and Absolute Positioning**

> "* **`position: relative;`** keeps the element in the normal document flow. Offset properties (`top`, `bottom`, `left`, `right`) push the element relative to where it *would normally sit\*, leaving a blank placeholder gap where it originally was.
>
> - **`position: absolute;`** completely removes the element from the normal document flow (taking up zero layout space). It positions itself relative to its closest parent that has a `position` other than `static` (typically a parent with `position: relative`)."

---

### **5) Difference Between `call()`, `apply()`, and `bind()**`

All three methods are used to explicitly set the `this` context for a function.

```javascript
const person = { name: "Alice" };

function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`);
}

// 1. call(): Invokes function immediately with arguments passed individually (comma-separated)
greet.call(person, "Hello", "!"); // Output: "Hello, Alice!"

// 2. apply(): Invokes function immediately with arguments passed as an Array
greet.apply(person, ["Hi", "."]); // Output: "Hi, Alice."

// 3. bind(): Returns a NEW bound function to be called later with specified context
const boundGreet = greet.bind(person, "Hey");
boundGreet("?"); // Output: "Hey, Alice?"
```

| Method        | Execution        | Argument Passing               | Return Value             |
| ------------- | ---------------- | ------------------------------ | ------------------------ |
| **`call()`**  | Immediately      | Arguments listed individually  | Result of function       |
| **`apply()`** | Immediately      | Arguments array `[arg1, arg2]` | Result of function       |
| **`bind()`**  | **Lazy / Later** | Arguments listed individually  | **A brand-new function** |

---

### **6) Remove Duplicate Values from an Array Without Using Built-In Methods**

To achieve this without built-in methods (like `Set`, `filter()`, `indexOf()`, `includes()`, or `reduce()`):

```javascript
function removeDuplicates(arr) {
  const result = [];
  const seen = {};

  for (let i = 0; i < arr.length; i++) {
    const currentItem = arr[i];

    // Check if item exists as a key in our tracking object
    if (!seen[currentItem]) {
      seen[currentItem] = true;
      result[result.length] = currentItem; // Push manually using array length index
    }
  }

  return result;
}

// Example Execution
const numbers = [1, 2, 3, 2, 4, 1, 5, 3];
console.log(removeDuplicates(numbers)); // Output: [1, 2, 3, 4, 5]
```

**How it works:**

- Uses a **Hash Map / Object (`seen`)** for $O(1)$ constant time lookup.
- Avoids `arr.push()` by writing directly to `result[result.length]`.
- Runs in **$O(n)$ time complexity** and **$O(n)$ space complexity**.

## Q. **What is Lazy Loading?**

**Lazy loading** is an optimization technique where non-essential application resources (such as code bundles, images, components, or modules) are deferred and loaded **on demand** (only when needed) rather than upfront during initial application startup.

In Angular, lazy loading primarily refers to **Route-Level Code Splitting**: instead of bundling the entire application into a single massive JavaScript file (`main.js`), Angular breaks the app into smaller feature bundles. The browser downloads a feature bundle only when the user navigates to its corresponding URL path.

---

### **Why is Lazy Loading Important?**

1. **Faster Initial Load Time (FCP & LCP):** Reduces the initial JavaScript bundle size that the browser needs to download, parse, and execute on first visit.
2. **Bandwidth Efficiency:** Users don't waste mobile data downloading pages or features they may never visit (e.g., an `/admin` dashboard for a regular guest user).
3. **Improved Resource Allocation:** Memory and CPU overhead stay low on startup.

---

### **How to Implement Lazy Loading in Angular**

#### **1. Modern Angular (v16+ Standalone Components)**

In modern Angular, lazy loading is achieved directly in the router configuration using the `loadComponent` or `loadChildren` functions with dynamic JavaScript imports (`import()`).

```typescript
// app.routes.ts
import { Routes } from "@angular/router";

export const routes: Routes = [
  {
    path: "",
    loadComponent: () =>
      import("./home/home.component").then((m) => m.HomeComponent),
  },
  {
    path: "dashboard",
    // Bundle for Dashboard is downloaded ONLY when visiting /dashboard
    loadComponent: () =>
      import("./dashboard/dashboard.component").then(
        (m) => m.DashboardComponent,
      ),
  },
  {
    path: "admin",
    // Lazy loads an entire feature route sub-tree
    loadChildren: () =>
      import("./admin/admin.routes").then((m) => m.ADMIN_ROUTES),
  },
];
```

#### **2. Legacy Angular (NgModule-based)**

In older versions of Angular, lazy loading was configured using `loadChildren` pointing to an `NgModule`.

```typescript
{
  path: 'orders',
  loadChildren: () => import('./orders/orders.module').then(m => m.OrdersModule)
}

```

---

### **Lazy Loading vs. Preloading Strategy**

While lazy loading defers downloading until navigation, you can optimize UX further using Angular **Preloading Strategies**:

- **Default (No Preloading):** Downloads feature bundles only when requested.
- **`PreloadAllModules`:** Downloads the initial bundle first so the page loads instantly, then automatically preloads all lazy-loaded bundles in the background while the user is idle.

```typescript
// app.config.ts
import {
  provideRouter,
  withPreloading,
  PreloadAllModules,
} from "@angular/router";
import { routes } from "./app.routes";

export const appConfig = {
  providers: [provideRouter(routes, withPreloading(PreloadAllModules))],
};
```

---

> _"Lazy loading is a performance strategy that defers the loading of code bundles, components, or assets until they are explicitly requested by the user. In Angular, we use dynamic imports (`loadComponent` or `loadChildren`) in our route definitions to perform code-splitting. This keeps our initial `main.js` bundle minimal, reducing initial page load times"_

## Q.

In Angular, Content Projection (historically called transclusion) is a pattern that allows you to insert (or "project") HTML content or components from a parent component into a designated slot inside a child component's template.

It is the primary way to build reusable, flexible UI components.

> \*"Content Projection in Angular is a mechanism that lets us pass HTML elements or components from a parent component into specific slots inside a child component using the `<ng-content>` element.

---

### **Key Rules to Remember for Interviews**

| Concept                              | Explanation                                                                                                                                                               |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`ng-content` is not DOM creation** | `<ng-content>` doesn't create DOM nodes; it acts purely as a placeholder or marker for where existing DOM nodes should land.                                              |
| **Lifecycle & Scope**                | Projected content is evaluated in the **parent component's context**, not the child's. Form controls, bindings, and events inside projected content belong to the parent. |
| **`ngProjectAs`**                    | Used when you need to project content into a `select` slot, but the projected element is wrapped inside a container like `<ng-container>`.                                |

---

## Q. You need to display 1000 records while maintaining good performance. How would you implement it?

> "To display 1,000 records efficiently without causing DOM lag, my primary choice is **CDK Virtual Scrolling** (`cdk-virtual-scroll-viewport`). It creates a virtualized window that renders only the ~15–20 rows visible in the viewport, keeping DOM node creation constant ($O(1)$) regardless of whether there are 1,000 or 1,000,000 records.
> If Virtual Scrolling isn't feasible due to variable row heights, I'd fall back to **Pagination / Infinite Scroll** paired with **`OnPush` Change Detection** and explicit `@for (track item.id)` tracking to minimize re-renders."
