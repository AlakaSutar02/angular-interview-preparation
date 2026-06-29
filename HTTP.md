# Angular HttpClient Interview Preparation

### Question 1: How do you perform a typed HTTP GET request using Angular's HttpClient?

**Expected Answer:**

You inject `HttpClient` and pass an interface type parameter to the generic method. This lets the TypeScript compiler know the exact shape of the returned Observable payload.

```typescript
interface User {
  id: number;
  name: string;
}

// Inside service:
this.http.get<User>('/api/users/1');
```

### Question 2: Why don't we typically need to manually unsubscribe from an `http.get()` stream inside a component?

**Expected Answer:**

Angular's `HttpClient` manages underlying browser network lifecycles. Once the server emits a response (or throws an error), the framework automatically invokes the stream's `.complete()` lifecycle sequence. A completed observable automatically cleans up and destroys its internal subscription list, preventing memory leaks.

### Question 3: How do you handle common HTTP errors on an individual request level?

**Expected Answer:**

You append the RxJS `catchError` operator to the HTTP observable pipe. It intercepts an `HttpErrorResponse`, allowing you to inspect status codes (e.g., 404, 500) and return a fallback value or re-throw a clean message using `throwError`.

---

_Focus: Interception pipelines, state mutation, and structural headers._

### Question 4: How do Functional Interceptors work in modern Angular, and how do they differ from legacy class-based interceptors?

**Expected Answer:**

Introduced in Angular 15, Functional Interceptors are standard standalone functions (`HttpInterceptorFn`) configured via `provideHttpClient(withInterceptors([...]))`. They leverage functional dependency injection (`inject()`) rather than needing a class architecture, making them:

- Cleaner to read
- Easier to write
- Tree-shakeable during production compilation

### Question 5: How do you append a Bearer Authorization token to every outgoing API request uniformly?

**Expected Answer:**

You use an interceptor to clone the incoming immutable request object and attach the headers, then pass the modified clone down the execution chain using `next()`.

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` },
  });
  return next(authReq);
};
```

### Question 6: What are HttpContext tokens, and when would you use them?

**Expected Answer:**

`HttpContext` tokens allow components/services to pass custom metadata along with an individual HTTP request that interceptors can read. For example, you can create a `BYPASS_LOGGING` token to tell a global logging interceptor to ignore a specific low-priority analytics request.

### Question 7: How do you architecture an Interceptor to handle a 401 Unauthorized token refresh without creating a request race condition?

**Expected Answer:**

When a 401 occurs, you catch the error and check a locking flag (`isRefreshing`).

**The Flow:**

1. The first request sets `isRefreshing = true` and triggers the token refresh API call.
2. Subsequent concurrent requests hit the else branch and are blocked/queued using an RxJS `BehaviorSubject` (or a replay mechanism).
3. Once the refresh succeeds, you update the subject, release the queued requests, and replay them with the updated token via a `switchMap`.

### Question 8: How would you design an Exponential Backoff Retry strategy for transient network drops (e.g., status 503 or 0)?

**Expected Answer:**

You chain the RxJS `retry` operator inside a global resilience interceptor. You configure the `delay` property of the operator to receive a callback function that:

- Tracks the `retryCount`
- Uses a `timer()` calculation to space out the attempts progressively (e.g., `retryCount * 1000ms`)
- Ensures you fail fast on structural client errors (like 400 or 403)
- Gives server blips room to recover

### Question 9: Why is Angular's HttpClient unsuitable for handling native Server-Sent Events (SSE) or WebSockets, and what should you use instead?

**Expected Answer:**

`HttpClient` is fundamentally built on top of the browser's `XMLHttpRequest` (XHR) API, which operates on a strict single-request, single-response transactional cycle.

For long-lived, multi-emission push streams, you should:

- Bypass `HttpClient` and instantiate the browser's native `EventSource` API (for SSE)
- Or wrap a native `WebSocket` inside a custom RxJS Observable state machine

### Question 3: How do you handle common HTTP errors on an individual request level?

- **Expected Answer:** You append the RxJS `catchError` operator to the HTTP observable pipe. It intercepts an `HttpErrorResponse`, allowing you to inspect status codes (e.g., 404, 500) and return a fallback value or re-throw a clean message using `throwError`.

---

_Focus: Interception pipelines, state mutation, and structural headers._

### Question 4: How do Functional Interceptors work in modern Angular, and how do they differ from legacy class-based interceptors?

- **Expected Answer:** Introduced in Angular 15, Functional Interceptors are standard standalone functions (`HttpInterceptorFn`) configured via `provideHttpClient(withInterceptors([...]))`. They leverage functional dependency injection (`inject()`) rather than needing a class architecture, making them cleaner to read, easier to write, and tree-shakeable during production compilation.

### Question 5: How do you append a Bearer Authorization token to every outgoing API request uniformly?

- **Expected Answer:** You use an interceptor to clone the incoming immutable request object and attach the headers, then pass the modified clone down the execution chain using `next()`.
  ```typescript
  export const authInterceptor: HttpInterceptorFn = (req, next) => {
    const token = inject(AuthService).getToken();
    const authReq = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
    return next(authReq);
  };
  ```

### Question 6: What are HttpContext tokens, and when would you use them?

**Answer:**  
`HttpContext` tokens allow components or services to pass custom, type-safe metadata along with an individual HTTP request that interceptors can subsequently read and act upon.

**Use Case:**  
You would use them to alter the behavior of global interceptors on a per-request basis. For example, you can create a `BYPASS_LOGGING` token to tell a global logging interceptor to ignore a specific low-priority analytics request.

---

### Question 7: How do you architect an Interceptor to handle a 401 Unauthorized token refresh without creating a request race condition?

**Answer:**  
To prevent race conditions when handling multiple concurrent requests that fail with a `401 Unauthorized` status, you must implement a locking and queuing mechanism inside your interceptor:

1. **Catch the Error & Lock:** When a `401` occurs, catch the error and check a locking boolean flag (e.g., `isRefreshing`).
2. **First Request Initiation:** The first request to hit the block sets `isRefreshing = true` and triggers the token refresh API call.
3. **Queue Concurrent Requests:** Subsequent concurrent requests that fail while a refresh is already in progress hit the `else` branch. They are blocked and queued using an RxJS `BehaviorSubject` (or a similar replay mechanism) holding a null/cached token state.
4. **Release and Replay:** Once the refresh mechanism succeeds, you emit the new token to the `BehaviorSubject`, release the queued requests, and replay them with the updated token via a `switchMap`.

---

### Question 8: How would you design an Exponential Backoff Retry strategy for transient network drops (e.g., status 503 or 0)?

**Answer:**  
You can chain the RxJS `retry` operator inside a global resilience interceptor to progressively delay retry attempts:

- **Configuration:** Configure the `delay` property of the `retry` operator to receive a callback function that tracks the current `retryCount`.
- **Calculation:** Use an RxJS `timer()` calculation to space out the attempts progressively (e.g., `retryCount * 1000ms` or $2^{\text{retryCount}} \times 1000\text{ms}$).
- **Conditional Filtering:** Ensure you check the error status code inside the callback so you fail fast on structural client errors (like `400` or `403`), while giving transient server blips (like `503 Service Unavailable` or `0` network drops) room to recover.

---

### Question 9: Why is Angular’s HttpClient unsuitable for handling native Server-Sent Events (SSE) or WebSockets, and what should you use instead?

**Answer:**  
Angular's `HttpClient` is fundamentally built on top of the browser’s `XMLHttpRequest` (XHR) API, which operates on a strict single-request, single-response transactional cycle. It cannot inherently sustain or stream persistent, open-ended connections.

**What to use instead:**  
For long-lived, multi-emission push streams, you must bypass `HttpClient`:

- **For SSE:** Instantiate the browser's native `EventSource` API.
- **For WebSockets:** Wrap a native `WebSocket` connection inside a custom RxJS `Observable` state machine or leverage specialized libraries like `rxjs/webSocket`.
