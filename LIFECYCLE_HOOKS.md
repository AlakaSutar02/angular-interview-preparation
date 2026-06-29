### Lifecycle hooks

In Angular, components and directives have a natural lifespan—they are created, they get updated when data changes, and eventually, they are destroyed when the user navigates away.

Lifecycle hooks are basically Angular's way of letting us plug our own code into specific moments of that lifespan.

I usually divide them into three main phases depending on the task:

Creation/Initialization: ngOnInit is my go-to for fetching initial data or setting up reactive logic.

Updates/Changes: If a component has @Input or signal properties changing over time, ngOnChanges lets me react immediately to those changes.

Destruction/Cleanup: ngOnDestroy is crucial for performance. It’s where I unsubscribe from Observables or clear timers so the app doesn't leak memory."

### The Hook Order :

"I think about the lifecycle order in three distinct chapters: Birth, Updates, and Death.

1. The Birth Phase (Initialization) :
   Before the component even renders its HTML, Angular has to set up its data.

ngOnChanges always goes first. Angular looks at any incoming data from the parent (@Input properties or signal inputs). If there's data coming in, it processes it here first.

ngOnInit runs right after. This is where the component is officially 'born' and ready. It only runs once, making it the perfect spot to fetch your API data or set up initial state.

2. The Content and View Phase (Rendering) :
   Once the data is ready, Angular starts rendering the layout. This happens from the inside out:

The 'Content' hooks (ngAfterContentInit / ngAfterContentChecked): These run first because Angular checks any external HTML you projected into the component (like using <ng-content>).

The 'View' hooks (ngAfterViewInit / ngAfterViewChecked): These run next. This means the component's own HTML template and all of its child elements are fully drawn and ready in the DOM. (Tip: If you need to touch the DOM directly, ngAfterViewInit is the milestone you wait for).

3. The Running/Update Phase (The Loop)
   While the user is interacting with the page, Angular enters a continuous checking loop.

Every time a user clicks something, types, or an API returns, Angular runs ngDoCheck to spot changes.

If data changed, it will re-trigger ngOnChanges and follow up with the Checked versions of the Content and View hooks (ngAfterContentChecked and ngAfterViewChecked) to make sure the UI matches the new data.

4. The Death Phase (Cleanup)
   Finally, when the user navigates away, ngOnDestroy fires right before the component is wiped from the DOM. This is your exit strategy to clean up subscriptions and prevent memory leaks."

### What Is the Difference Between ngOnInit & constructor?

The Constructor (The "Delivery Guy") :

"The constructor is a standard TypeScript feature. In Angular, its main superpower is Dependency Injection. I use it almost exclusively to say, 'Hey Angular, I need this HTTP service and this router service for this component to function.'

It is a best practice to keep the constructor completely empty of business logic. You shouldn't be making API calls or calculating values here because, at this exact moment, the component isn't fully ready yet."

ngOnInit (The "Grand Opening") :

"On the other hand, ngOnInit is an Angular lifecycle hook. It fires after the constructor has finished and after Angular has successfully bound data to your component's @Input or signal properties.

Because everything is hooked up and available, this is the perfect 'safe zone' to kick off heavy initialization tasks—like fetching data from your backend or setting up complex form groups."
