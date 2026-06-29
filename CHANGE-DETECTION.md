When Angular updates your application, it does so in a strict, single-pass flow:

Pass 1 (Change Detection): Angular looks at your component data, updates the HTML view, and remembers the values it used.

Pass 2 (Verification Loop): In development mode, Angular immediately runs a second check to see if any values changed during or immediately after that first pass.

If Angular finds that a value changed between Pass 1 and Pass 2, it throws this error. It is essentially saying: "Hey! I just finished rendering the page, but code somewhere just snuck in and changed the data behind my back. If this happens in production, your UI might show wrong or out-of-sync data."

A Common Example: The Child-to-Parent Side Effect
The most common cause is a child component modifying its parent's data while the parent is still in the middle of rendering.

The Problematic Code
Imagine a parent component that displays a status message, and a child component that updates that message inside its ngOnInit or lifecycle hooks.

Parent Component:

```
@Component({
  selector: 'app-parent',
  standalone: true,
  imports: [ChildComponent],
  template: `
    <!-- Angular reads 'status' here during Pass 1 -->
    <h1>Status: {{ status }}</h1>
    <app-child (loaded)="status = 'Child is ready!'"></app-child>
  `
})
export class ParentComponent {
  status = 'Loading...';
}
```

Why it fails:
Pass 1: Angular checks ParentComponent. It reads status as 'Loading...' and prints it to the screen.

Angular moves down to check ChildComponent.

The child's ngOnInit runs and fires loaded.emit().

The parent receives this and changes status to 'Child is ready!'.

Pass 2: Angular verifies the app. It looks at the parent's status again and finds it is now 'Child is ready!' instead of 'Loading...'. Boom. Error thrown.

How to Fix It
There are three common ways to fix this, depending on your architecture:

Fix 1: Microtask Deferral (The Quick Fix)
You can force the change to happen in the next JavaScript event loop cycle using setTimeout or asapScheduler. This gives Angular time to finish its entire verification pass before the value changes.

```
// Inside the Child Component
ngOnInit() {
  setTimeout(() => {
    this.loaded.emit();
  }, 0);
}
```

Fix 2: Use the Correct Lifecycle Hook
If you are changing a value inside a component based on its own internal layout, make sure you aren't doing it inside ngAfterViewInit. Move it to ngOnInit if possible, because ngOnInit runs before the DOM template is officially rendered and locked in.

Fix 3: Use Signals (Modern Angular Solution)
If you are using Angular 17+, Signals handle dependency tracking much cleaner. If your state is driven reactively via computed signals or explicit user actions rather than cyclical lifecycle hooks, you naturally avoid breaking the unidirectional data flow.
