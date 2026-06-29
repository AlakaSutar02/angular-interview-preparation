# Angular Routing Guide

## Overview

Routing helps you change what the user sees in a single-page app. **Angular Router** (`@angular/router`) is the official library for managing navigation in Angular applications.

### Initial Setup

When bootstrapping an Angular application without the Angular CLI, you can pass a configuration object that includes a `providers` array. Inside this array, add the Angular router by calling the `provideRouter` function with your routes:

```typescript
bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes)]
});
```

## Core Routing Concepts

### 1. **Routes** (The Configuration Map)

An array of objects that defines your application's layout strategy. It tells Angular: *"When the user types this path into the browser, load this specific component."*

```typescript
const routes: Routes = [
  { path: 'home', component: HomeComponent },
  { path: 'profile', component: ProfileComponent },
  { path: '**', component: NotFoundComponent } // Wildcard route
];
```

**Important:** When you define routes, the **order is critical** because Angular uses a **first-match-wins strategy**. Wildcard routes (`**`) must be placed at the end of the array.

---

### 2. **Router Outlet** (`<router-outlet>`)

A declarative placeholder directive inside your HTML template. Think of it as a dynamic viewport or a portal where Angular mounts and unmounts components matching the active route.

```html
<nav>
  <a routerLink="/home">Home</a>
  <a routerLink="/profile">Profile</a>
</nav>

<router-outlet></router-outlet>
```

---

### 3. **Router Link** (`routerLink`)

The attribute directive used in your HTML instead of a standard `href`. 

| Feature | `href="/profile"` | `routerLink="/profile"` |
|---------|-------------------|------------------------|
| Behavior | Forces full browser reload | Intercepts click, updates only necessary components |
| Performance | Slower (entire page reload) | Faster (SPA navigation) |

```html
<a routerLink="/profile" [queryParams]="{ id: 123 }">View Profile</a>
```

---

### 4. **ActivatedRoute** & **RouteSnapshot**

#### ActivatedRoute
- An injectable service representing the currently loaded route
- Exposes data (parameters, query strings) as **RxJS Observables**
- Use when a component stays loaded but the route parameters change

```typescript
constructor(private route: ActivatedRoute) {}

ngOnInit() {
  this.route.params.subscribe(params => {
    console.log('User ID:', params['id']);
  });
}
```

#### ActivatedRouteSnapshot
- A static, immutable read-only picture of route data at a single moment in time
- Contains **no observables**
- Use when you only need the data once during component initialization

```typescript
constructor(private route: ActivatedRoute) {}

ngOnInit() {
  const userId = this.route.snapshot.params['id'];
}
```

#### Common ActivatedRoute Properties

| Property | Type | Description |
|----------|------|-------------|
| `url` | Observable | Route paths as an array of strings |
| `data` | Observable | Data object provided for the route + resolved values |
| `params` | Observable | Required and optional parameters specific to the route |
| `queryParams` | Observable | Query parameters available to all routes |

---

### 5. **Route Guards**

Lightweight functional interceptors that act as automated security gates. They evaluate conditions (e.g., "Is this user logged in?") to allow or block a navigation request before any code is rendered.

**Types of Guards:**
- `canActivate` - Prevent navigation to a route
- `canDeactivate` - Prevent leaving a route
- `canLoad` - Prevent lazy-loaded modules from loading
- `canActivateChild` - Protect child routes

```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  return authService.isLoggedIn() ? true : inject(Router).createUrlTree(['/login']);
};

const routes: Routes = [
  { path: 'admin', component: AdminComponent, canActivate: [authGuard] }
];
```

---

### 6. **Data Resolvers**

Middleware functions that fetch required backend API data during the routing process. They hold back the visual transition until the data arrives, preventing empty-screen layout shifts.

```typescript
export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  return userService.getUser(route.params['id']);
};

const routes: Routes = [
  { 
    path: 'profile/:id', 
    component: ProfileComponent,
    resolve: { user: userResolver }
  }
];
```

---

## Loading Strategies

Angular offers two primary strategies to control how and when components are loaded:

### Eagerly Loaded Routes

Routes and components that are loaded immediately when the application starts.

**Pros:**
- Components available instantly when user navigates
- Seamless transitions with no loading delays

**Cons:**
- Larger initial bundle size
- Slower Time to Interactive (TTI)
- Users download code they might never use

```typescript
const routes: Routes = [
  { path: 'home', component: HomeComponent },
  { path: 'about', component: AboutComponent }
];
```

---

### Lazy Loaded Routes

Routes and components loaded **only when needed** - i.e., when the user navigates to them.

**Why Use Lazy Loading:**
- ✅ Reducing initial bundle sizes
- ✅ Improving Core Web Vitals (Largest Contentful Paint, First Input Delay)
- ✅ Saving end-user data consumption
- ✅ Faster initial page load

#### The Problem: The "Mega-Bundle"

By default, when you build a standard Angular application, the framework bundles all components, services, and HTML templates into massive JavaScript files. 

**Example:** Imagine a corporate portal with:
- Login Page (used by everyone)
- User Dashboard (used by everyone)  
- Admin Control Panel (used by 2% of users)
- Analytics Suite with heavy graphing libraries (used once a month)

Without lazy loading, a regular employee opening the login page downloads **all** of this code, including the admin panel and analytics suite they'll never use. With lazy loading, they only download what they need.

#### Implementing Lazy Loading

```typescript
const routes: Routes = [
  { path: 'login', component: LoginComponent },
  { 
    path: 'admin', 
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canActivate: [adminGuard]
  },
  { 
    path: 'analytics', 
    loadChildren: () => import('./analytics/analytics.module').then(m => m.AnalyticsModule)
  }
];
```

---

## Angular Router Lifecycle

The router follows **4 core phases**:

### 1. **Navigation Start & Resolution**
The router intercepts the URL and figures out where it needs to go.

### 2. **Guarding** (The Security Checkpoint)
Running the guard chain to authorize or halt the transition.

### 3. **Resolving** (Data Pre-fetching)
Fetching asynchronous data before the component even thinks about rendering.

### 4. **Activation & Finalization**
Destroying old components, instantiating new ones, and updating the browser history.

---

## Common Interview Questions & Answers

### Q: How do you debug router lifecycle issues in production vs. development?

**A:** 
- **Development:** Pass `withDebugTracing()` to `provideRouter` (or `enableTracing: true` in legacy `RouterModule.forRoot`)
- **Production:** Write a custom service that subscribes to `router.events` to log navigation changes

```typescript
// Development
bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes, withDebugTracing())]
});

// Production - Custom logging
constructor(private router: Router) {
  this.router.events.subscribe(event => {
    if (event instanceof NavigationStart) console.log('Navigation started');
    if (event instanceof NavigationEnd) console.log('Navigation ended');
  });
}
```

---

### Q: What happens to an active HTTP request in a Resolver if the user clicks 'Back' mid-transition?

**A:** The router doesn't automatically cancel HTTP requests inside resolvers unless you explicitly tie the RxJS lifecycle. The navigation will emit `NavigationCancel`, but to stop the backend load, you need to manually unsubscribe or use the `takeUntil` operator with a destroy subject:

```typescript
export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  const router = inject(Router);
  
  return userService.getUser(route.params['id']).pipe(
    takeUntil(
      router.events.pipe(
        filter(event => event instanceof NavigationCancel)
      )
    )
  );
};
```

---

## Summary Table

| Concept | Purpose | Key Feature |
|---------|---------|------------|
| Routes | Define path-to-component mappings | First-match-wins strategy |
| Router Outlet | Placeholder for rendered components | Dynamic viewport |
| Router Link | Navigate without page reload | Intercepts clicks |
| ActivatedRoute | Access current route data as Observables | Dynamic updates |
| RouteSnapshot | Access route data once at initialization | Immutable snapshot |
| Guards | Control navigation access | Security gates |
| Resolvers | Pre-fetch data before rendering | Prevent empty states |
| Lazy Loading | Load modules on demand | Reduce bundle size |
