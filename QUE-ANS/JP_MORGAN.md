## Q. What are Interceptors?

In Angular, an **Interceptor** is a mechanism that intercepts incoming HTTP requests and outgoing responses processed by Angular's `HttpClient`.

Think of it as **middleware for network requests**. It allows you to inspect, transform, or modify HTTP calls globally before they are sent to the server, or before their responses reach your components/services.

---

### **Primary Use Cases**

1. **Authentication & Authorization:** Automatically attach JWT bearer tokens to outgoing `Authorization` headers.
2. **Global Error Handling:** Intercept 401 (Unauthorized), 403 (Forbidden), or 500 (Server Error) status codes globally to show notifications or redirect to login.
3. **Logging & Analytics:** Log performance metrics (e.g., request duration) or log network failures to an internal server.
4. **Loading Spinners:** Automatically increment/decrement an active HTTP request counter to display a global loading spinner.
5. **Caching:** Return cached responses directly for repeated GET requests to avoid unnecessary network calls.

---

### **Modern Angular Example (Functional Interceptors - Angular 15+)**

In modern Angular (v15+), functional interceptors using `HttpInterceptorFn` are preferred over legacy class-based interceptors (`HttpInterceptor`).

#### **1. Creating a Functional Auth Interceptor (`auth.interceptor.ts`)**

```typescript
import { HttpInterceptorFn } from "@angular/common/http";
import { inject } from "@angular/core";
import { AuthService } from "./auth.service";

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  // Clone request because HttpRequest objects in Angular are immutable
  if (token) {
    const clonedRequest = req.clone({
      setHeaders: {
        Authorization: `Bearer ${token}`,
      },
    });
    return next(clonedRequest);
  }

  // Pass request untransformed if no token exists
  return next(req);
};
```

#### **2. Registering Interceptors in `app.config.ts**`

```typescript
import { ApplicationConfig } from "@angular/core";
import { provideHttpClient, withInterceptors } from "@angular/common/http";
import { authInterceptor } from "./auth.interceptor";

export const appConfig: ApplicationConfig = {
  providers: [
    // Registers HTTP client with the functional interceptor chain
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
};
```

---

### **Key Rules to Remember in Interviews**

- **Immutability:** `HttpRequest` and `HttpResponse` objects are immutable. You cannot modify them directly; you must clone them using `req.clone()`.
- **Chaining:** Interceptors execute in the exact order they are declared in the `withInterceptors([...])` array for outgoing requests, and in **reverse order** for incoming responses.

## Q. Difference between setValue() and patchValue()

In Angular Reactive Forms, both **`setValue()`** and **`patchValue()`** are used to update the values of a `FormGroup` or `FormArray`, but they differ in strictness and how they enforce the form structure.

---

### **Key Difference Summary**

| Feature                   | `setValue()`                                                               | `patchValue()`                                                           |
| ------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| **Structure Enforcement** | **Strict.** The object must match the exact shape of the `FormGroup`.      | **Flexible.** Accepts partial objects and ignores missing properties.    |
| **Missing Control Error** | **Throws an error** if any control key is missing from the payload.        | **Fails silently** (updates available keys and ignores missing ones).    |
| **Extra Control Error**   | **Throws an error** if the payload contains key names not in the form.     | Ignores any extra keys not present in the `FormGroup`.                   |
| **Primary Use Case**      | Whole form resets/resets from API endpoints where structure is guaranteed. | Partial updates, single field updates, or dynamic/optional API payloads. |

---

### **Code Example Comparison**

Assume we have the following `FormGroup`:

```typescript
this.userForm = new FormGroup({
  name: new FormControl(""),
  email: new FormControl(""),
  age: new FormControl(null),
});
```

#### **1. Using `setValue()` (Strict)**

Must pass values for **all** form controls (`name`, `email`, and `age`):

```typescript
// ✅ WORKS: All keys are present
this.userForm.setValue({
  name: "John Doe",
  email: "john@example.com",
  age: 30,
});

// ❌ THROWS RUNTIME ERROR: "Must supply a value for form control with name: 'age'."
this.userForm.setValue({
  name: "John Doe",
  email: "john@example.com",
});
```

#### **2. Using `patchValue()` (Flexible)**

Can pass partial updates without throwing errors:

```typescript
// ✅ WORKS: Only updates 'email', leaves 'name' and 'age' unchanged
this.userForm.patchValue({
  email: "newemail@example.com",
});

// ✅ WORKS: Extra keys like 'address' are safely ignored
this.userForm.patchValue({
  name: "Jane Doe",
  address: "123 Main St",
});
```

---

> "The main difference between `setValue()` and `patchValue()` in Reactive Forms is strictness:
>
> - **`setValue()`** is strict. It requires you to supply an object that matches the exact structure of the `FormGroup`. If any control key is missing or extra, Angular throws a runtime error. It’s ideal when you want to enforce strict schema validation—for instance, when populating a form with a complete data model from a backend API.
> - **`patchValue()`** is flexible. It allows partial updates, modifying only the specified controls while leaving the rest unchanged without throwing errors. It’s best suited for single-field updates, dynamic forms, or multi-step form workflows."
