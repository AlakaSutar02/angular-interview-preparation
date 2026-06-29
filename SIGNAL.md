# Angular Signals: Complete Guide

## Why Signals Were Introduced

For years, Angular relied on **Zone.js** for dirty-checking. Zone.js monkey-patches all asynchronous browser APIs (like `setTimeout`, `fetch`, or click events). When any async event fires, Zone.js tells Angular to check the entire component tree for changes.

### The Problem with Zone.js
In massive enterprise apps, this top-down checking causes major performance bottlenecks. Developers had to use:
- `NgZone.runOutsideAngular()` 
- Heavy `OnPush` configurations

### The Signal Solution
Signals introduce **fine-grained reactivity**. Instead of checking the whole tree, Angular now knows exactly which specific DOM node depends on which specific piece of state. This eliminates unnecessary re-renders and improves performance dramatically.

---

## Signal Types

### 1. Writable Signals (`signal()`)

A signal is a getter function that holds a value and notifies consumers when that value changes.

**Usage:**
```typescript
state = signal(initialValue);
```

**Functional Update Approach:**
```typescript
this.count.update(c => c + 1);
```

Use the functional update approach to safely modify signals based on their previous state.

---

### 2. Computed Signals (`computed()`)

Read-only derived state that is **memoized** and **lazily evaluated**.

**Key Features:**
- **Memoization**: If source signals don't change, the computation doesn't run
- **Dynamic Dependency Tracking**: Automatically drops signals that are no longer referenced (e.g., in branching if/else blocks)
- **Prevents Unnecessary Recalculations**: Only recomputes when tracked signals actually change

**Example Use Case:**
```typescript
// Computed signal that derives from other signals
filteredItems = computed(() => this.items().filter(i => i.active));
```

---

### 3. Effects (`effect()`)

An operation that runs whenever its tracking signals change.

**Important Guidelines:**
- Effects are **strictly for side effects**, not state synchronization
- If you find yourself writing to a writable signal inside an effect, use a **computed signal** instead
- Effects must be used responsibly to avoid infinite loops or unnecessary re-runs

**Use Effects For:**
- Logging changes
- Syncing with external systems
- Triggering analytics

---

## Signals vs. RxJS

### Signals Do NOT Replace RxJS

**RxJS Remains Essential For:**
- Asynchronous operations
- Orchestration and race condition handling
- API polling
- Operators like `switchMap`, `debounceTime`, and complex pipelines

### Signals Excel At:
- **Synchronous State Representation**: Completely synchronous, glitch-free (eliminates the diamond problem)
- **Template Layer**: Require absolutely no boilerplate (no async pipes needed)
- **Fine-grained Reactivity**: Automatic change tracking

---

## Architectural Interoperability: RxJS ↔ Signals

Bridge the gap between Observables and Signals using `@angular/core/rxjs-interop`:

### `toSignal(observable$)`
Converts an Observable to a Signal.

**Key Benefit:**
- Automatically handles unsubscription when the context is destroyed
- Eliminates memory leak risks natively

```typescript
data = toSignal(this.apiService.getData$());
```

### `toObservable(signal)`
Converts a Signal into an Observable stream, allowing you to pipe operators.

**Use Case:**
```typescript
// Pipe debounceTime and switchMap onto signal changes
searchResults$ = toObservable(this.searchQuery).pipe(
  debounceTime(300),
  switchMap(query => this.apiService.search(query))
);
```

---

## Recommended Architecture Pattern

### Hybrid Model: Best of Both Worlds

**Data Layer (Services):**
- Use **RxJS** for HTTP streams, error handling, and WebSockets
- Leverage powerful asynchronous pipelines

**Component Layer (UI):**
- Expose data using `toSignal()`
- Use **Writable and Computed Signals** exclusively
- Eliminate async pipe boilerplate

**Benefits:**
- ✅ Retain powerful asynchronous capabilities of RxJS
- ✅ Incredibly clean templates with Signals
- ✅ No async pipe verbosity
- ✅ Automatic change detection optimization
- ✅ Type-safe reactive state

### Example Architecture

```typescript
// Service Layer - RxJS
@Injectable()
export class DataService {
  data$ = this.http.get('/api/data');
}

// Component Layer - Signals
export class MyComponent {
  dataService = inject(DataService);
  
  // Convert Observable to Signal for template
  data = toSignal(this.dataService.data$, { initialValue: [] });
  
  // Create derived state with Computed
  filteredData = computed(() => this.data().filter(...));
}
```

---

## Interview Tips

### Key Points to Mention:
1. **Problem it solves**: Replace Zone.js dirty-checking with fine-grained reactivity
2. **Performance**: Eliminates unnecessary re-renders in large applications
3. **Hybrid approach**: Keep RxJS for async operations, Signals for UI state
4. **Interoperability**: Show knowledge of `toSignal()` and `toObservable()`
5. **Memory management**: Automatic unsubscription prevents leaks

### Example Answer to "How are you incorporating Signals?"
> "I adopt a hybrid model where RxJS handles asynchronous operations in the service layer (HTTP, WebSockets, error handling), while components expose that data through `toSignal()`. In the UI layer, I exclusively use Writable and Computed Signals. This gives us the best of both worlds—powerful async capabilities with incredibly clean, performant templates that need no async pipes."
