### 1. What is NGRX and Why Use It?

ngrx is a state management library for Angular that uses the principles of Redux.

## 1. What is NgRx, and what problem does it solve?

_The foundational question to test if you understand the "why" behind global state management._

> "NgRx is a framework for building reactive applications in Angular, inspired by the Redux pattern. It provides a single, immutable, global **Store** that acts as the single source of truth for the entire application.
> It solves three major problems in large apps:
>
> 1. **Shared State:** Eliminates complex parent-child component communication (`@Input`/`@Output`) or messy shared services.
> 2. **Predictability:** State can only be mutated in a strict, unidirectional flow, making debugging straightforward.
> 3. **Performance:** By using immutable state, we can switch our components to `ChangeDetectionStrategy.OnPush`, significantly reducing unnecessary template re-renders."

---

## 2. Explain the core pillars of NgRx. How does data flow?

_This tests your knowledge of the unidirectional loop._

The flow moves in a strict, un-breachable circle:

- **Store:** The single database container holding your application's current state. It is read-only.
- **Actions:** Unique events that describe _something happened_ in the application (e.g., `[User Page] Click Login Button`). Components dispatch actions.
- **Reducers:** Pure functions responsible for handling state transitions. They listen to Actions, take the _current_ state, copy it, apply changes, and return a _brand new_ state object.
- **Selectors:** Pure functions used by components to slice, filter, and read specific pieces of state from the Store.
- **Effects:** An asynchronous middleware layer. They listen for Actions, perform side-effects (like fetching data from an HTTP API Service), and then dispatch a _new_ Action containing the result (e.g., `LoadUserSuccess`).

---

## 3. What is the difference between NgRx Store, NgRx ComponentStore, and SignalStore?

_Crucial for modern Angular roles to show you are up to date._

- **NgRx Store (Global Store):** Best for global application state shared across completely different modules (e.g., User Authentication, global themes, shared shopping carts). It involves writing actions, reducers, and selectors.
- **NgRx ComponentStore:** Designed to manage local, independent component state (e.g., a complex data table with its own pagination, sorting, and loading states). It dies when the component is destroyed.
- **NgRx SignalStore (Modern):** Introduced to align with Angular Signals. It is a fully functional, lightweight, signal-based state management solution. It removes almost all traditional Redux boilerplate code, providing native reactivity without forcing developers to use RxJS streams in their components.

---

## 4. How do you optimize NgRx performance in large applications?

_Show the interviewer you have built enterprise-level code._

1. **Memoized Selectors:** Always use `createSelector` to build selectors. They cache their results. If the underlying state hasn't changed, the selector doesn't recalculate, preventing laggy UI components.
2. **Feature States (Lazy Loading):** Do not load your entire application state inside `app.config.ts`. Use `provideState()` and `provideEffects()` inside lazy-loaded routes so that code bundles and states are only initialized when the user enters that module.
3. **Smart vs. Dumb Components:** Only "Smart" container components should inject the Store to dispatch actions or read selectors. "Dumb" presentational components should just receive data via inputs and emit events via outputs, keeping the component tree clean and fast.

---

## 5. Senior Pro-Tip: "Now that we have Angular Signals, do we still need NgRx?"

_Expect this question. Deliver a balanced architectural response._

> "Yes, absolutely. Signals and NgRx solve different problems.
> **Signals** are a state _primitive_—they manage local reactivity inside a component beautifully and tell the DOM exactly what to redraw.
> However, Signals on their own do not enforce architectural rules. In a massive enterprise app with dozens of developers, relying _only_ on local signals leads to scattered, unpredictable state mutations. We still need **NgRx** to enforce a strict unidirectional data flow, provide global state debugging tools (Redux DevTools), handle complex async race conditions via Effects, and maintain a clear separation of concerns."
