# Angular Templates: Complete Guide

## What Are Templates in Angular?

In Angular, templates are **not** parsed at runtime as raw HTML string manipulation. Instead, they are strictly typed, declarative blueprints.

### Compilation Process

During the build phase, the **Ivy compiler** transforms these templates into highly optimized JavaScript instructions. This means:
- Template bindings map directly to DOM creation
- Change detection is optimized at compile time
- Zero runtime overhead from template parsing

---

## Built-In Control Flow (Angular 17+)

### Modern Replacement for Structural Directives

Angular has completely revolutionized template syntax with **Built-In Control Flow** (`@if`, `@for`, `@switch`), moving away from legacy structural directives (`*ngIf`, `*ngFor`).

### Key Advantages:

✅ **Zero Imports Required** - No need to import `CommonModule` in standalone components  
✅ **Reduced Bundle Size** - No directive overhead in the final bundle  
✅ **Perfect Type Narrowing** - Inside `@if` blocks, TypeScript knows the narrowed type  
✅ **Mandatory Fallback** - `@else` block ensures all code paths are covered  

### Syntax Examples:

**@if Block:**
```typescript
@if (isLoggedIn) {
  <p>Welcome, {{ userName }}</p>
} @else if (isPending) {
  <p>Loading...</p>
} @else {
  <p>Please log in</p>
}
```

**@for Block:**
```typescript
@for (item of items; track item.id) {
  <div>{{ item.name }}</div>
}
```

**@switch Block:**
```typescript
@switch (status) {
  @case ('pending') {
    <p>Processing...</p>
  }
  @case ('success') {
    <p>Completed!</p>
  }
  @default {
    <p>Unknown status</p>
  }
}
```

---

## Deferrable Views (@defer)

### Revolutionary Performance Tuning from the Template

One of the most powerful features available in modern Angular templates is **Deferrable Views** (`@defer`). It allows us to declaratively lazy-load expensive UI components directly from the HTML layer.

### Key Features:

🚀 **Lazy Loading** - Components load only when needed  
📊 **Core Web Vitals** - Dramatically improves initial page-load performance  
🎯 **Declarative States** - Manage loading, error, and placeholder states  
⚡ **Completely Changes Performance Strategy** - Managed entirely through template blocks  

### Declarative State Blocks:

```typescript
@defer (on viewport) {
  <heavy-component />
} @placeholder {
  <p>Loading component...</p>
} @loading (after 500ms) {
  <loading-spinner />
} @error {
  <p>Failed to load component</p>
}
```

### Trigger Options:

- `on viewport` - Load when element enters viewport
- `on interaction` - Load on user interaction (click, hover)
- `on timer(ms)` - Load after specified milliseconds
- `on immediate` - Load right away (but separately from main bundle)

---

## Template Type Checking

### Ensuring Strict Maintainability

Working across large teams, strict maintainability is key. Always enable `strictTemplates` in `tsconfig.json`:

```json
{
  "angularCompilerOptions": {
    "strictTemplates": true,
    "strictInputTypes": true,
    "strictOutputEventTypes": true,
    "strictAttributeTypes": true
  }
}
```

### Benefits:

✅ Angular type-checks templates against component classes during build  
✅ Catches binding errors before runtime  
✅ Prevents typos in property/event names  
✅ Full IDE autocompletion and refactoring support  

---

## Data Binding Mechanics

### 1. Interpolation `{{ value }}`

One-way data binding from component state to the DOM.

```typescript
// Component
name = 'Angular';

// Template
<p>Hello {{ name }}</p>  // Output: Hello Angular
```

---

### 2. Property Binding `[property]="value"`

Sets DOM elements or component properties.

```typescript
// Component
isDisabled = false;

// Template
<button [disabled]="isDisabled">Click me</button>
<img [src]="imageUrl" [alt]="imageAlt" />
```

**Senior Tip:** Always emphasize binding to properties rather than DOM attributes for performance consistency.

---

### 3. Event Binding `(event)="handler()"`

Flows data from the DOM back to the component class.

```typescript
// Component
count = 0;
increment() {
  this.count++;
}

// Template
<button (click)="increment()">Clicks: {{ count }}</button>
```

---

### 4. Two-Way Binding `[(ngModel)]` or `model()`

Synchronizes state both ways automatically.

**Legacy Approach (ngModel):**
```typescript
<input [(ngModel)]="userName" />
```

**Modern Approach (Signal model()):**
```typescript
// Component
userName = model<string>('');

// Template
<input [(ngModel)]="userName()" />
```

**Senior Tip:** In modern architectures, prefer using the new `model()` inputs for two-way binding. It bypasses the complexity of ngModel and provides better type safety.

---

## DOM & Template Interaction Primitives

### 1. Template Reference Variables (`#var`)

Grab a reference to a DOM element, component instance, or directive within the template.

```typescript
// Template
<input #userInput type="text" />
<button (click)="processInput(userInput.value)">Submit</button>

// Component
processInput(value: string) {
  console.log(value);
}
```

---

### 2. `<ng-template>`

Define structural fragments that aren't rendered by default. Essential for advanced content projection and conditional rendering.

```typescript
<ng-template #customHeader>
  <h1>Custom Header Content</h1>
</ng-template>

@if (showCustomHeader) {
  <ng-container *ngTemplateOutlet="customHeader" />
}
```

---

### 3. `<ng-container>`

A logical wrapper that doesn't introduce extra nodes to the final DOM structure.

```typescript
// Component
layout = 'grid';

// Template
<ng-container [ngSwitch]="layout">
  @case ('grid') {
    <div class="grid-layout">
      <item-a />
      <item-b />
    </div>
  }
  @case ('flex') {
    <div class="flex-layout">
      <item-a />
      <item-b />
    </div>
  }
</ng-container>
```

**Lead Tip:** Strictly use `<ng-container>` to apply layout logic or control flows without muddying the DOM tree with empty `<div>` tags. This keeps our CSS flex/grid structures intact.

---

## Interview Answer: Template Design & Optimization

### Sample Response:

> "I treat templates as a strictly typed extension of our business logic. I build them using modern Built-In Control Flow (`@if`, `@for`) to gain maximum compilation efficiency, and I always enforce `strictTemplates: true` in my projects.
>
> For performance-critical apps, I heavily leverage Deferrable Views (`@defer`) to break up monolithic feature modules, ensuring that heavy below-the-fold UI components are only fetched when the user actually needs them. This dramatically improves initial page-load performance and Core Web Vitals scores.
>
> I also prioritize:
> - Using `<ng-container>` for structural logic to keep the DOM clean
> - Template reference variables for DOM interactions
> - Signal-based two-way binding (`model()`) for modern state management
> - Semantic HTML and accessibility standards throughout"

---

## Key Takeaways for Interviews

✅ Modern Built-In Control Flow replaces legacy structural directives  
✅ Deferrable Views enable declarative lazy-loading for performance  
✅ `strictTemplates: true` catches errors at build time  
✅ Understand all data binding mechanics: interpolation, property, event, two-way  
✅ Use template primitives correctly: `#var`, `<ng-template>`, `<ng-container>`  
✅ Treat templates as typed, optimized code, not just HTML strings
