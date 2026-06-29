# Angular Directives: Enterprise Architecture & Interview Preparation

An in-depth guide to Angular directives, their architectural role, and senior-level patterns for building extensible, reusable UI behaviors.

---

## 📌 Core Definition

**A Directive is an instruction to the DOM.**

Directives allow us to attach declarative, reusable behavior to native DOM elements or custom components without polluting our core business logic. With modern Angular, directives are written as **Standalone by default**, allowing them to be imported dynamically only where needed, keeping bundles highly tree-shaken.

---

## 🏗️ Three Pillars of Directives

### 1. Attribute Directives

#### The Concept
Changing the behavior, appearance, or layout of an existing DOM element.

#### Senior Highlight
Direct DOM manipulation vs. Angular abstractions (Renderer2, HostListener, HostBinding).

#### How to Voice It

> "Attribute directives modify the behavior or styling of an element. When building these, I strictly avoid direct DOM manipulation via `ElementRef.nativeElement.style` because it breaks platform abstraction and causes issues with Server-Side Rendering (SSR).
>
> Instead, I leverage `@HostBinding` and `@HostListener`—or in modern Angular, the `host` property metadata within the `@Directive` decorator—paired with `Renderer2`. This keeps our DOM manipulations safe, platform-agnostic, and compatible with all Angular environments."

---

### 2. Structural Directives

#### The Concept
Shaping or reshaping the DOM's structure by adding, removing, or manipulating elements.

#### Senior Highlight
Understanding `<ng-template>`, `ViewContainerRef`, and `TemplateRef`.

#### How to Voice It

> "Structural directives are responsible for layout manipulation. They are easily recognized by the asterisk (`*`) syntax, which is syntactic sugar for wrapping an element in an `<ng-template>`.
>
> Under the hood, a structural directive injects `TemplateRef` (the blueprint of what to render) and `ViewContainerRef` (the container where it will be rendered).
>
> While modern Angular's built-in control flow (`@if`, `@for`) has replaced the need for legacy structural directives like `*ngIf`, creating custom structural directives—such as an `*appHasRole` directive for role-based rendering—remains a powerful pattern for enterprise applications requiring complex conditional logic."

---

### 3. Components

#### The Concept
Directives with a template. (Components are technically just directives with associated views.)

> **Note:** Components are the most feature-rich directive type. They have templates, styles, and lifecycle hooks. See the COMPONENT.md guide for detailed coverage.

---

## 🔗 Directive Composition API (Angular 15+)

### The Problem: Multi-Directive Inheritance

Historically, if a component needed the behavior of three different directives, you had to manually declare all of them on the host element—leading to boilerplate and tight coupling.

### The Solution: Declarative Composition

> "One of the most powerful features added to Angular recently is the **Directive Composition API**. Now, using the `hostDirectives` property in a component or directive configuration, we can declaratively compose directives. We can inject behaviors—like a tracking directive, a tooltip directive, and a validation directive—directly into a single host without polluting the template.
>
> This is a critical pattern for enterprise design systems and framework-level abstractions."

#### Example: Composing Multiple Directives

```typescript
import { Directive, Component } from '@angular/core';

// Individual directives
@Directive({
  selector: '[appTracking]',
  standalone: true
})
export class TrackingDirective {
  // Tracking logic
}

@Directive({
  selector: '[appTooltip]',
  standalone: true
})
export class TooltipDirective {
  // Tooltip logic
}

// Compose them together
@Component({
  selector: 'app-button',
  template: `<button>Click me</button>`,
  standalone: true,
  hostDirectives: [TrackingDirective, TooltipDirective]
})
export class ButtonComponent {}
```

---

## 🎯 Master Plan: Directive Excellence for Senior Interviews

### 1. The Definitive Overview

Establish the core definition, but pivot immediately to how directives are fundamentally categorized in modern Angular.

> "At its core, a **Directive is an instruction to the DOM**. In fact, **Components are technically just directives with templates**.
>
> From an architectural standpoint, Directives are our primary tool for **enforcing separation of concerns**. They allow us to attach declarative, reusable behavior to native DOM elements or custom components without polluting our core business logic.
>
> With modern Angular, I write directives as **Standalone by default**, allowing them to be imported dynamically only where needed, keeping our bundles highly tree-shaken and our dependency graph clean."

---

### 2. Categorizing the Three Pillars of Directives

Clearly delineate the types of directives, focusing on how and when to use them at scale.

#### Attribute Directives

**Best For:**
- Styling or behavior modifications
- Cross-cutting concerns (analytics, validation, accessibility)
- Reusable UI enhancements

**Example Pattern:**
```typescript
@Directive({
  selector: '[appHighlight]',
  standalone: true,
  host: {
    '[style.background-color]': 'backgroundColor',
    '(mouseenter)': 'onHover(true)',
    '(mouseleave)': 'onHover(false)'
  }
})
export class HighlightDirective {
  backgroundColor = 'yellow';
  
  onHover(isHovering: boolean) {
    this.backgroundColor = isHovering ? 'gold' : 'yellow';
  }
}
```

#### Structural Directives

**Best For:**
- Conditional rendering based on complex logic
- Layout transformations
- Role-based or permission-based UI rendering

**Example Pattern:**
```typescript
@Directive({
  selector: '[appHasRole]',
  standalone: true
})
export class HasRoleDirective {
  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private authService: AuthService
  ) {}

  @Input()
  set appHasRole(requiredRole: string) {
    if (this.authService.hasRole(requiredRole)) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}
```

#### Components

The highest-level directive type with template and styling integration. Use for isolated, reusable UI blocks with their own state and lifecycle.

---

## 🎨 Host Elements: The Modern Syntax

### Legacy Approach
```typescript
@Directive({
  selector: '[appTooltip]',
  standalone: true
})
export class TooltipDirective {
  @HostBinding('class.has-tooltip') hasTooltip = true;
  
  @HostListener('mouseenter')
  showTooltip() { /* ... */ }
  
  @HostListener('mouseleave')
  hideTooltip() { /* ... */ }
}
```

### Modern Preference
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

#### Why the Modern Approach Wins

- **Cleaner and more declarative** - All host interactions are visible at a glance
- **Groups concerns together** - All element interactions are colocated at the top of the file
- **Better for optimization** - Plays nicer with performance optimization and signals
- **Easier to refactor** - Single source of truth for host behavior

---

## 💼 Enterprise Application Patterns

### Using Directives to Solve Architectural Challenges

#### The Philosophy

> "I use directives primarily to **keep components clean and focused** purely on their designated business logic. If I see UI behaviors creeping into components—like custom analytics tracking, input masking, intersection observers, or accessibility enhancements—those belong in directives.
>
> I build them using `Renderer2` and `host` metadata to ensure they are **safe for Server-Side Rendering (SSR)**. Furthermore, in our design systems, I heavily leverage the **Directive Composition API**. It allows our teams to build composable, reusable behaviors that scale across enterprise applications without coupling."

#### Practical Example: Cross-Cutting Concerns

```typescript
// Analytics tracking directive
@Directive({
  selector: '[appTrack]',
  standalone: true,
  host: {
    '(click)': 'trackClick()'
  }
})
export class TrackDirective {
  @Input() appTrack: string = '';
  
  constructor(private analytics: AnalyticsService) {}
  
  trackClick() {
    this.analytics.event('user_interaction', {
      element: this.appTrack
    });
  }
}

// Use it on any element
// <button appTrack="submit-button">Submit</button>
```

---

## 🎓 Interview Key Takeaways

### What an Interviewer Wants to Hear

1. **Understand the hierarchy** - Components are directives with templates; directives are instructions to the DOM
2. **Know the three types** - Attribute, Structural, and Component directives have distinct purposes
3. **Demonstrate modern patterns** - Use `host` metadata, Standalone directives, and Composition API
4. **Show architectural thinking** - Explain how directives keep applications maintainable and scalable
5. **Avoid footguns** - Know why direct DOM manipulation is problematic; always use Renderer2 and platform abstractions
6. **SSR awareness** - Understand that directives must be compatible with Server-Side Rendering

### Red Flags to Avoid

- ❌ Direct manipulation: `ElementRef.nativeElement.style = '...'`
- ❌ NgModule-only directives in modern code
- ❌ Over-use of structural directives when modern control flow (`@if`, `@for`) suffices
- ❌ Ignoring SSR compatibility in enterprise applications

---

## 📚 Related Topics

- **[COMPONENT.md](./COMPONENT.md)** - Component lifecycle, change detection, and memory management
- **[DI.md](./DI.md)** - Dependency Injection patterns for injecting services into directives
- **[http.md](./http.md)** - Using HTTP interceptors alongside directives for cross-cutting concerns

---

## 🔗 Resources

- [Angular Directives Documentation](https://angular.dev/guide/directives)
- [Directive Composition API](https://angular.dev/guide/directives/directive-composition-api)
- [Angular Style Guide - Directives](https://angular.dev/style-guide#directives)

---

**Last Updated:** June 2026
**Version:** Angular 21+
