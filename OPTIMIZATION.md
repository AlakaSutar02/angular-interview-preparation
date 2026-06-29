1. Runtime & Change Detection (The Performance Boosters)
   ChangeDetectionStrategy.OnPush: "By default, Angular checks the entire component tree whenever anything changes. Switching components to OnPush tells Angular to only re-render that component if its input references change or an event fires explicitly. It completely eliminates unnecessary background checks."

trackBy with Loops: "When rendering lists with \*ngFor (or using the modern @for loop), always use a trackBy function or tracking property. This prevents Angular from tearing down and rebuilding the entire DOM tree every time the array updates; it only updates the specific row that changed."

Cleaning Up Unused Subscriptions: "To prevent major memory leaks, it's vital to unsubscribe from infinite RxJS observables when a component dies. I do this using takeUntilDestroyed in modern Angular, or by leveraging the async pipe in templates which cleans up after itself automatically."

2. Bundle Size & Loading (The Fast-Load Tactics)
   Lazy Loading: "Instead of forcing the user to download the entire application code on day one, we should use lazy loading. In modern Angular, we use deferred views (@defer) or lazy route configurations to only load feature components exactly when the user navigates to them."

AOT (Ahead-of-Time) Compilation: "AOT compilation compiles the HTML templates into highly optimized JavaScript during the build process rather than at runtime in the browser. It’s enabled by default now, but it's essential for keeping the runtime footprint minimal."

3. Build & Dependency Cleanliness
   Auditing Dependencies: "I regularly use tools like webpack-bundle-analyzer to visually inspect what's inside the production bundle. It helps catch duplicate utility libraries or large packages that shouldn't be there, allowing us to drop unused imports and keep dependencies lean."

### webpack-bundle-analyzer

1. Install the Analyzer
   Install webpack-bundle-analyzer as a development dependency:

Bash
npm install --save-dev webpack-bundle-analyzer

# OR

yarn add -D webpack-bundle-analyzer 2. Generate the Build Stats
Run the Angular production build command with the --stats-json flag. This instructs the Angular compiler to output a stats.json file inside your build output folder (usually dist/<project-name>).

Bash
ng build --configuration production --stats-json 3. Run the Analyzer
Execute the analyzer command and point it directly to the generated stats.json file:

Bash
npx webpack-bundle-analyzer dist/<your-project-name>/stats.json
Note: Replace <your-project-name> with the actual folder name inside your dist/ directory.

Once executed, a local web server will automatically open in your browser (usually at [http://127.0.0.1:8888](http://127.0.0.1:8888)), displaying a zoomable tree map of your application bundles.
