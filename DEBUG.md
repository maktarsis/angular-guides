# Debugging Angular Applications

- `ng.probe($0).componentInstance` (browser script) <br/> <br/>
This is a quirky command that we can use on the console to see what the component’s state is. Simply select the component you want to inspect from the Elements tab and execute ng.probe($0).componentInstance on the console. $0 is a global variable the Chrome DevTools make available, whose value is the most recently selected element. You can use this for $1, $2, $3 & $4 to inspect the previous four DOM elements. 

- `ng.profiler.timeChangeDetection()` (browser script) <br/> <br/>
Angular has a built-in profiler which can be used to profile your Angular application. At the moment, it offers only one kind of profiling — timeChangeDetection. timeChangeDetection performs change detection on the application and prints how long a round of change detection takes for the current state of UI. The recommended value for this is to be less than 3 milliseconds. <br/> <br/>
You can also record this profiling for further analysis by passing in an argument. <br/>
`ng.profiler.timeChangeDetection({ record: true })` <br/>
To start using `ng.profiler.timeChangeDetection()` you have to include it in project <br/>
`main.ts`

```typescript
// NgModuleRef & ApplicationRef
import { enableProdMode, NgModuleRef, ApplicationRef } from '@angular/core';
// important module for debugging
import { enableDebugTools } from '@angular/platform-browser';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';
import { environment } from './environments/environment';
if (environment.production) {
  enableProdMode();
}
platformBrowserDynamic().bootstrapModule(AppModule)
  // here we enable debug tools after bootstrap app
  .then((module: NgModuleRef<AppModule>) => {
    if (environment.production) {
      return;
    }
    // Profiler for application
    // ng.profiler.timeChangeDetection()
    const applicationRef = module.injector.get(ApplicationRef);
    const appComponent = applicationRef.components[0];
    enableDebugTools(appComponent);
  })
  .catch(err => console.log(err));
```

- **_tap()_** (RXJS Library) <br/> <br/>
If you are using RxJs operators, you might find that debugging the operator chain can be a bit tricky and having a .tap() between the operators makes it easier to inspect the chain. It gives us the ability to watch the chain without actually modifying it.
<br/>
tap is the pipeable operator equivalent of do in RxJs version 5.5 and above. The name was changed to avoid conflicts with Javascript keyword do.

```typescript
const wholeNumbers$ = of(1, 2, 3, 4, 5);
const squareNumbers$ = wholeNumbers$
.pipe(
  tap(num => console.log(`Whole number: ${num}`)),
  map(num => num * num),
  tap(num => console.log(`Square number: ${num}`))
);
```

<img src="https://cdn-images-1.medium.com/max/800/1*7FyXQpaGTntUIfimj-Hgpw.png" alt="result"/>

- <a href="https://augury.angular.io/">**_Augury_**</a> (Chrome extension)
<br/> <br/>
Provides a visual representation of the Angular components on a page, their dependencies, router information, whether change detection is used for the component, etc.
<br/>
<img src="https://cdn-images-1.medium.com/max/800/1*WxUaE0LoRkGQYNfaw3gb3A.jpeg" alt="example"/>

- **_Redux_** (Chrome extension)
<br/> <br/>
  If you are using Redux to manage your application state, there is a Chrome extension called Redux DevTools which lets us see all the actions dispatched, state of the application after each action and the difference in states.
<br/>

<img src="/assets/ngrx-devtool-example.gif?raw=true" alt="example"/>