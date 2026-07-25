Handling errors in Angular effectively requires a tiered strategy. You need to catch errors gracefully in your UI, intercept network failures, and have a global net to catch anything that slips through.

Here is the clean, production-grade blueprint for handling errors in modern Angular.

---

## 1. The Global Catch-All (`ErrorHandler`)

By default, when an unhandled exception occurs, Angular logs it to the browser console, but the app might freeze. You can override this behavior by implementing a custom `ErrorHandler`.

This is the perfect place to integrate logging tools (like Sentry or LogRocket) or show a global error notification.

### Create the Custom Handler:

```typescript
import { ErrorHandler, Injectable, NgZone, inject } from "@angular/core";

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private zone = inject(NgZone);

  handleError(error: any): void {
    // 1. Log the error to your backend server or Sentry
    console.error("Captured by Global Error Handler:", error);

    // 2. Alert the user (wrapped in NgZone so the UI updates instantly)
    this.zone.run(() => {
      alert("An unexpected error occurred. Our team has been notified!");
    });
  }
}
```

### Register it globally (`app.config.ts`):

```typescript
import { ApplicationConfig, ErrorHandler } from "@angular/core";

export const appConfig: ApplicationConfig = {
  providers: [{ provide: ErrorHandler, useClass: GlobalErrorHandler }],
};
```

---

## 2. Catching Network Errors Globally (`HttpInterceptor`)

You shouldn't write repetitive error-handling code for every single API call. Instead, use a functional **HTTP Interceptor** to intercept responses and manage standard status codes (like `401 Unauthorized` or `500 Server Error`) in one spot.

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from "@angular/common/http";
import { catchError, throwError } from "rxjs";

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = "An unknown error occurred!";

      if (error.error instanceof ErrorEvent) {
        // Client-side network or javascript error
        errorMessage = `Client Error: ${error.error.message}`;
      } else {
        // Backend API error codes
        switch (error.status) {
          case 401:
            errorMessage = "Session expired. Please log in again.";
            break;
          case 403:
            errorMessage =
              "You do not have permission to access this resource.";
            break;
          case 404:
            errorMessage = "The requested resource was not found.";
            break;
          case 500:
            errorMessage = "Internal Server Error. Please try again later.";
            break;
        }
      }

      // Pass the error back down to the component if it wants to handle it locally too
      return throwError(() => new Error(errorMessage));
    }),
  );
};
```

---

## 3. Local Component Error Handling (RxJS vs. Signals)

Depending on whether you use RxJS or modern Signals in your components, local error handling looks slightly different:

### Using RxJS Streams:

Use the `catchError` operator or provide an `error` callback blocks inside your subscription block:

```typescript
this.userService.getProfile().subscribe({
  next: (data) => this.user.set(data),
  error: (err) => this.localErrorMessage.set(err.message), // Handle locally
});
```

### Using Modern Signals:

Because computed signals are evaluated synchronously, an error inside a computing block will break the view unless wrapped cleanly in a native `try/catch` block:

```typescript
// Wrapping dynamic operations safely
userData = computed(() => {
  try {
    const rawData = this.rawSignalData();
    return this.transformData(rawData);
  } catch (err) {
    this.errorState.set("Failed to parse data");
    return null;
  }
});
```

---

## 4. how you structure error handling, structure your answer like this:

> 1. **Network Layer:** I use a functional `HttpInterceptor` to catch status codes globally, manage refresh tokens on `401` errors, and format server responses.
> 2. **Application Layer:** I implement a custom `ErrorHandler` provider to capture runtime JavaScript exceptions and seamlessly pipe them to external telemetry systems like Sentry.
> 3. **UI Layer:** I leverage local component states or conditional templates (`@if` blocks) to cleanly swap broken components out for placeholder "Error/Retry" states, maintaining a smooth user experience.
