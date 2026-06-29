### The Connection to Dependency Injection (DI):

"In Angular, components don’t create their own services. Instead, they just say, 'Hey, I need the HttpClient service.' Angular's DI system looks at the Providers registered in the app to figure out how to instantiate that service and hand it over."

The "No Provider" Rule:

"For every dependency you want to inject, Angular must have a provider registered somewhere. If you try to inject a service without a provider, Angular won't know how to build it and will throw a No provider for... error."

Where You Register Them (Scope):

"The cool thing about providers is that where you register them changes how they behave. There are three main ways to do it:

Root Level (providedIn: 'root'): This makes the service a global singleton available everywhere, which is the modern Angular standard.

Component Level: If you put a provider inside a specific component's @Component({ providers: [...] }) array, Angular creates a fresh, isolated instance of that service only for that component and its children.

Module Level: (In older or non-standalone apps) It scopes the service to a specific NgModule."
