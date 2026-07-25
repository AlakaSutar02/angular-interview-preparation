### **Q1. "What's the difference between `providedIn: 'root'` and providing in a module's `providers: []`?"**

> **How to answer:**
> "`providedIn: 'root'` makes a service **tree-shakable**. If no component or service imports or injects it, the compiler completely drops it from the final JavaScript bundle.
> When you declare a service inside an `NgModule`'s `providers: []` array, it forces the service to be included in the bundle as long as that module is loaded, whether anything uses it or not. Both register the service in the Root Environment Injector as an app-wide singleton, but `providedIn: 'root'` is the modern standard because it keeps our bundle size optimized."

**Example:**

```typescript
// Modern, Tree-Shakable (Preferred)
@Injectable({
  providedIn: "root",
})
export class UserService {}

// Legacy / Non Tree-Shakable (Avoid for global singletons)
@NgModule({
  providers: [UserService], // Included as long as module is loaded
})
export class UserModule {}
```

---

### **Q2. "How do you inject a value that isn't a class?"**

> **How to answer:**
> "You use an `InjectionToken<T>`. Classes work natively as DI lookup keys because TypeScript transpiles them to JS constructor functions. Non-class values—like primitive strings, configuration objects, or API endpoints—don't exist as JS objects at runtime.
> An `InjectionToken` creates a unique runtime key that we can pair with `useValue` or `useFactory`."

**Example:**

```typescript
export interface AppConfig {
  apiUrl: string;
}

// 1. Define the runtime token
export const APP_CONFIG = new InjectionToken<AppConfig>("APP_CONFIG", {
  factory: () => ({ apiUrl: "https://api.example.com" }),
});

// 2. Inject it in a component/service
@Injectable({ providedIn: "root" })
export class ApiService {
  private config = inject(APP_CONFIG); // Reads the configuration token
}
```

---

### **Q3. "Where can `inject()` be called?"**

> **How to answer:**
> "`inject()` can only be called inside an **Injection Context**. That means class field initializers, constructor parameters, factory functions, or framework-invoked context like Route Guards, Resolvers, and Interceptors.
> You cannot call `inject()` inside regular class methods or asynchronous callbacks (like a `.subscribe()` or `setTimeout`) because by the time those run, Angular's internal injector pointer has already moved on. If you need a dependency in a method, you capture it in a field during class instantiation."

**Example:**

```typescript
@Component({ ... })
export class UserComponent {
  //  VALID: Field initializer (Injection Context)
  private userService = inject(UserService);

  constructor() {
    //  VALID: Constructor body (Injection Context)
    const logger = inject(LoggerService);
  }

  onClick() {
    // ❌ INVALID: Will throw "NG0203: inject() must be called from an injection context"
    const http = inject(HttpClient);
  }
}

```

---

### **Q4. "What happens if two providers provide the same token?"**

> **How to answer:**
> "It depends on where they are declared in the injector hierarchy:
>
> 1. **At the same level:** The last provider declared in the array overrides any previous ones ('last one wins').
> 2. **At different levels:** Angular searches bottom-up (Element Injector $\rightarrow$ Route Injector $\rightarrow$ Root Injector). The injector nearest to the requesting component wins.
>
> The major exception is using `{ multi: true }`—for tokens like `HTTP_INTERCEPTORS` or `APP_INITIALIZER`, where Angular merges all providers into an array instead of overwriting them."

**Example:**

```typescript
// Last one wins at the same injector level
providers: [
  { provide: LoggerService, useClass: ConsoleLogger },
  { provide: LoggerService, useClass: FileLogger }, // FileLogger overrides ConsoleLogger
];
```

---

### **Q5. "How do you get a fresh service instance per component?"**

> **How to answer:**
> "You declare the service in the component's own `@Component({ providers: [...] })` metadata.
> This registers the service in that component's **Element Injector**. Every time Angular instantiates that component, it creates a brand-new instance of the service scoped strictly to that component and its child sub-tree. When the component is destroyed, the service instance is garbage collected."

**Example:**

```typescript
@Component({
  selector: "app-widget",
  templateUrl: "./widget.component.html",
  providers: [WidgetStateService], // Fresh instance for every <app-widget> on page
})
export class WidgetComponent {
  // Each instance gets its own isolated state service
  public widgetState = inject(WidgetStateService);
}
```

---

### **Q6. "How does DI compose with lazy routes?"**

> **How to answer:**
> "Lazy routes create a dedicated **Environment Injector** when the route is loaded. Services provided in a route's `providers: []` array are singletons within that route's sub-tree.
> They can inject root-level singletons, but root-level singletons cannot inject route-scoped services. Once you navigate away from that route sub-tree, those route-scoped services are destroyed."

**Example:**

```typescript
export const ADMIN_ROUTES: Route[] = [
  {
    path: '',
    component: AdminComponent,
    providers: [AdminStateService], // Isolated singleton for all child routes under /admin
    children: [ ... ]
  }
];

```

---

### **Q7 (Trap). "Can you inject a service into a plain standalone function?"**

> **How to answer:**
> "Yes, **if** that plain function is executed within an active Injection Context—like a functional route guard, functional interceptor, or a custom `computed()` signal expression.
> If you need to invoke a plain helper function outside an active context (e.g., inside an event listener), you must explicitly wrap it using `runInInjectionContext(injector, fn)`."

**Example:**

```typescript
// 1. Functional Route Guard (Angular executes this in an Injection Context automatically)
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService); //  Works!
  return authService.isAuthenticated();
};

// 2. Manual Injection Context for arbitrary plain functions
function logUser(injector: Injector) {
  runInInjectionContext(injector, () => {
    const userSvc = inject(UserService); //  Works!
    console.log(userSvc.getCurrentUser());
  });
}
```

---

### **Q8. "What does `importProvidersFrom()` do?"**

> **How to answer:**
> "`importProvidersFrom()` is a migration bridge utility for standalone applications. It extracts all providers registered inside legacy `NgModule`s (and their imported modules) and transforms them into a clean `Provider[]` array that standalone `bootstrapApplication` or route-level `providers` can consume.
> You use it while migrating legacy third-party libraries (like NgRx or Angular Material) before they offer native `provideX()` standalone functions."

**Example:**

```typescript
// app.config.ts (Standalone App Bootstrap)
export const appConfig: ApplicationConfig = {
  providers: [
    // Extracts module-based providers into standalone bootstrap configuration
    importProvidersFrom(HttpClientModule, StoreModule.forRoot({})),
  ],
};
```

---

### **Q9. "How do you override a provider in a unit test?"**

> **How to answer:**
> "For Root or Environment providers, you override them in `TestBed.configureTestingModule({ providers: [...] })`.
> For component-level element providers (declared inside `@Component({ providers: [] })`), you must use `TestBed.overrideComponent()` to replace the provider array before calling `TestBed.createComponent()`."

**Example:**

```typescript
// Overriding an Element Injector provider on a component
TestBed.overrideComponent(UserComponent, {
  set: {
    providers: [{ provide: UserService, useValue: mockUserService }],
  },
});
```

---

### **Q10. "How does `providedIn: 'root'` interact with lazy modules?"**

> **How to answer:**
> "The service is registered in the **Root Environment Injector**, not in the lazy module's local injector.
> That means even if a service is only imported by code inside a lazy-loaded route, `providedIn: 'root'` still makes it a global app-wide singleton. It won't create duplicate instances per lazy route. If you want a service scoped specifically to a lazy boundary, you must provide it explicitly in the route definition or module providers."

---

### **Q11. "How would you provide different implementations in dev vs. prod?"**

> **How to answer:**
> "For compile-time differences, we use Angular's `fileReplacements` in `angular.json` to swap environment files (`environment.ts` vs `environment.prod.ts`).
> For runtime choices, we use a **Factory Provider** (`useFactory`) that inspects runtime flags or configuration services dynamically."

**Example:**

```typescript
export const LOGGER_PROVIDER: Provider = {
  provide: LoggerService,
  useFactory: () => {
    const env = inject(EnvConfigService);
    return env.isProduction ? new SentryLogger() : new ConsoleLogger();
  },
};
```

---

### **Q12 (Advanced). "You have a shared `NotificationService` singleton, but two lazy routes each want their own separate instance. How?"**

> **How to answer:**
> "The cleanest approach is to remove `providedIn: 'root'` from `NotificationService` and declare `providers: [NotificationService]` directly inside the route definitions of both lazy routes.
> Each lazy route creates its own Environment Injector boundary, so Route A and Route B will each instantiate their own separate, isolated instance of `NotificationService`."

**Example:**

```typescript
// 1. Class declared WITHOUT providedIn: 'root'
@Injectable()
export class NotificationService {}

// 2. Route Definitions
export const routes: Route[] = [
  {
    path: "orders",
    providers: [NotificationService], // Dedicated Instance #1
    loadChildren: () => import("./orders/orders.routes"),
  },
  {
    path: "billing",
    providers: [NotificationService], // Dedicated Instance #2
    loadChildren: () => import("./billing/billing.routes"),
  },
];
```
