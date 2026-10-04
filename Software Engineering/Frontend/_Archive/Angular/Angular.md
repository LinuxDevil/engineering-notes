---
title: "Angular"
---
[Interview Questions Angular](/Software%20Engineering/Interviews/Interview%20Questions%20Angular)


## What is zone?

A Zone is an execution context that persists across async tasks. Angular relies on zone.js to run Angular's change detection processes when native JavaScript operations raise events

What is NgZone?
    Angular provides a service called NgZone which creates a zone named `angular` to automatically trigger change detection when the following conditions are satisfied.
	1. When a sync or async function is executed.
	2. When there is no microTask scheduled.


What is NoopZone?
    Zone is loaded/required by default in Angular applications and it helps Angular to know when to trigger the change detection. This way, it make sures developers focus on application development rather core part of Angular. You can also use Angular without Zone but the change detection need to be implemented on your own and `noop zone` need to be configured in bootstrap process. Let's follow the below two steps to remove zone.js,
	1. Remove the zone.js import from polyfills.ts.
```typescript
/***************************************************************************************************
 - Zone JS is required by default for Angular itself.
 */
// import 'zone.js/dist/zone';  // Included with Angular CLI.
```
	2. Bootstrap Angular with noop zone in src/main.ts.
```plain text
platformBrowserDynamic().bootstrapModule(AppModule, {ngZone: 'noop'})
  .catch(err => console.error(err));
```

## What is folding?

The compiler can only resolve references to exported symbols in the metadata. Where as some of the non-exported members are folded while generating the code. i.e Folding is a process in which the collector evaluate an expression during collection and record the result in the .metadata.json instead of the original expression. For example, the compiler couldn't refer selector reference because it is not exported

```typescript
let selector = 'app-root';
@Component({
  selector: selector
})
```

Will be folded into inline selector

```typescript
@Component({
      selector: 'app-root'
    })
```

Remember that the compiler can’t fold everything. For example, spread operator on arrays, objects created using new keywords and function calls.

## What is JIT?

Just-in-Time (JIT) is a type of compilation that compiles your app in the browser at runtime. JIT compilation was the default until Angular 8, now default is AOT. When you run the ng build (build only) or ng serve (build and serve locally) CLI commands, the type of compilation (JIT or AOT) depends on the value of the aot property in your build configuration specified in angular.json. By default, aot is set to true.

### What is AOT?

Ahead-of-Time (AOT) is a type of compilation that compiles your app at build time. This is the default starting in Angular 9. When you run the ng build (build only) or ng serve (build and serve locally) CLI commands, the type of compilation (JIT or AOT) depends on the value of the aot property in your build configuration specified in angular.json. By default, aot is set to true.

The AOT compiler works in three phases,
	1. **Code Analysis:** The compiler records a representation of the source
	2. **Code generation:** It handles the interpretation as well as places restrictions on what it interprets.
	3. **Validation:** In this phase, the Angular template compiler uses the TypeScript compiler to validate the binding expressions in templates.

You can control your app compilation in two ways,
	1. By providing template compiler options in the `tsconfig.json` file
	2. By configuring Angular metadata with decorators

Below are the list of AOT benefits,
	1. **Faster rendering:** The browser downloads a pre-compiled version of the application. So it can render the application immediately without compiling the app.
	2. **Fewer asynchronous requests:** It inlines external HTML templates and CSS style sheets within the application javascript which eliminates separate ajax requests.
	3. **Smaller Angular framework download size:** Doesn't require downloading the Angular compiler. Hence it dramatically reduces the application payload.
	4. **Detect template errors earlier:** Detects and reports template binding errors during the build step itself
	5. **Better security:** It compiles HTML templates and components into JavaScript. So there won't be any injection attacks.


## Questions

1. The main building blocks of an Angular application are shown in the diagram below:-

![Untitled 43](/Attachments/Untitled%2043.png)

1. What are the key components of Angular?Angular has the key components below,
	1. **Component:** These are the basic building blocks of an Angular application to control HTML views.
	2. **Modules:** An Angular module is a set of angular basic building blocks like components, directives, services etc. An application is divided into logical pieces and each piece of code is called as "module" which perform a single task.
	3. **Templates:** These represent the views of an Angular application.
	4. **Services:** Are used to create components which can be shared across the entire application.
	5. **Metadata:** This can be used to add more data to an Angular class.

2. What are directives?
	Directives add behaviour to an existing DOM element or an existing component instance.
```plain text
import { Directive, ElementRef, Input } from '@angular/core';

@Directive({ selector: '[myHighlight]' })
export class HighlightDirective {
    constructor(el: ElementRef) {
       el.nativeElement.style.backgroundColor = 'yellow';
    }
}
```
    Now this directive extends HTML element behavior with a yellow background as below
```typescript
<p myHighlight>Highlight me!</p>
```

3. What are components?Components are the most basic UI building block of an Angular app, which form a tree of Angular components. These components are a subset of directives. Unlike directives, components always have a template, and only one component can be instantiated per element in a template. Let's see a simple example of Angular component
```plain text
import { Component } from '@angular/core';

@Component ({
   selector: 'my-app',
   template: ` <div>
      <h1>{{title}}</h1>
      <div>Learn Angular6 with examples</div>
   </div> `,
})

export class AppComponent {
   title: string = 'Welcome to Angular world';
}
```


4. What is a template?
	A template is a HTML view where you can display data by binding controls to properties of an Angular component. You can store your component's template in one of two places. You can define it inline using the template property, or you can define the template in a separate HTML file and link to it in the component metadata using the @Component decorator's templateUrl property.
	**Using inline template with template syntax,**
```typescript
import { Component } from '@angular/core';

@Component ({
   selector: 'my-app',
   template: '
      <div>
         <h1>{{title}}</h1>
         <div>Learn Angular</div>
      </div>
   '
})

export class AppComponent {
   title: string = 'Hello World';
}
```
    **Using separate template file such as app.component.html**
```plain text
import { Component } from '@angular/core';

@Component ({
   selector: 'my-app',
   templateUrl: 'app/app.component.html'
})

export class AppComponent {
   title: string = 'Hello World';
}
```

5. What is a module?
	Modules are logical boundaries in your application and the application is divided into separate modules to separate the functionality of your application. Lets take an example of **app.module.ts** root module declared with **@NgModule** decorator as below,
```typescript
import { NgModule }      from '@angular/core';
 import { BrowserModule } from '@angular/platform-browser';
 import { AppComponent }  from './app.component';

@NgModule ({
   imports:      [ BrowserModule ],
   declarations: [ AppComponent ],
   bootstrap:    [ AppComponent ],
   providers: []
})
 export class AppModule { }
```
    The NgModule decorator has five important (among all) options:
	1. The imports option is used to import other dependent modules. The BrowserModule is required by default for any web based angular application.
	2. The declarations option is used to define components in the respective module.
	3. The bootstrap option tells Angular which Component to bootstrap in the application.
	4. The providers option is used to configure a set of injectable objects that are available in the injector of this module.
	5. The entryComponents option is a set of components dynamically loaded into the view.

6. What are lifecycle hooks available?
	Angular application goes through an entire set of processes or has a lifecycle right from its initiation to the end of the application. The representation of lifecycle in pictorial representation as follows,
![lifecycle](/Attachments/lifecycle.png)
    The description of each lifecycle method is as below,
	6. **ngOnChanges:** When the value of a data bound property changes, then this method is called.
	7. **ngOnInit:** This is called whenever the initialization of the directive/component after Angular first displays the data-bound properties happens.
	8. **ngDoCheck:** This is for the detection and to act on changes that Angular can't or won't detect on its own.
	9. **ngAfterContentInit:** This is called in response after Angular projects external content into the component's view.
	10. **ngAfterContentChecked:** This is called in response after Angular checks the content projected into the component.
	11. **ngAfterViewInit:** This is called in response after Angular initializes the component's views and child views.
	12. **ngAfterViewChecked:** This is called in response after Angular checks the component's views and child views.
	13. **ngOnDestroy:** This is the cleanup phase just before Angular destroys the directive/component.

7. What is metadata?Metadata is used to decorate a class so that it can configure the expected behavior of the class. The metadata is represented by decorators
	14. **Class decorators**, e.g. @Component and @NgModule
```typescript
import { NgModule, Component } from '@angular/core';

@Component({
  selector: 'my-component',
  template: '<div>Class decorator</div>',
})
 export class MyComponent {
  constructor() {
    console.log('Hey I am a component!');
  }
}

@NgModule({
  imports: [],
  declarations: [],
})
 export class MyModule {
  constructor() {
    console.log('Hey I am a module!');
  }
}
```
	15. **Property decorators** Used for properties inside classes, e.g. @Input and @Output
```typescript
import { Component, Input } from '@angular/core';

@Component({
    selector: 'my-component',
    template: '<div>Property decorator</div>'
})

 export class MyComponent {
    @Input()
    title: string;
}
```
	16. **Method decorators** Used for methods inside classes, e.g. @HostListener
```typescript
import { Component, HostListener } from '@angular/core';

@Component({
    selector: 'my-component',
    template: '<div>Method decorator</div>'
})
 export class MyComponent {
    @HostListener('click', ['$event'])
    onHostClick(event: Event) {
        // clicked, `event` available
    }
}
```
	17. **Parameter decorators** Used for parameters inside class constructors, e.g. @Inject, @Optional
```plain text
import { Component, Inject } from '@angular/core';
import { MyService } from './my-service';

@Component({
    selector: 'my-component',
    template: '<div>Parameter decorator</div>'
})
export class MyComponent {
    constructor(@Inject(MyService) myService) {
        console.log(myService); // MyService
    }
}
```

8. What is the difference between constructor and ngOnInit?
	The **Constructor** is a default method of the class that is executed when the class is instantiated and ensures proper initialisation of fields in the class and its subclasses. Angular, or better Dependency Injector (DI), analyses the constructor parameters and when it creates a new instance by calling new MyClass() it tries to find providers that match the types of the constructor parameters, resolves them and passes them to the constructor.
	**ngOnInit** is a life cycle hook called by Angular to indicate that Angular is done creating the component.
	Mostly we use ngOnInit for all the initialization/declaration and avoid stuff to work in the constructor. The constructor should only be used to initialize class members but shouldn't do actual "work". So you should use constructor() to setup Dependency Injection and not much else. ngOnInit() is better place to "start" - it's where/when components' bindings are resolved.
```plain text
export class App implements OnInit{
  constructor(private myService: MyService){
     //called first time before the ngOnInit()
  }

  ngOnInit(){
     //called after the constructor and called  after the first ngOnChanges()
     //e.g. http call...
  }
}
```

9. What is a service?
	A service is used when a common functionality needs to be provided to various modules. Services allow for greater separation of concerns for your application and better modularity by allowing you to extract common functionality out of components.
	Let's create a repoService which can be used across components,
```typescript
import { Injectable } from '@angular/core';
 import { Http } from '@angular/http';

@Injectable({ // The Injectable decorator is required for dependency injection to work
  // providedIn option registers the service with a specific NgModule
  providedIn: 'root',  // This declares the service with the root app (AppModule)
})
 export class RepoService{
  constructor(private http: Http){
  }

  fetchAll(){
    return this.http.get('https://api.github.com/repositories');
  }
}
```
    The above service uses Http service as a dependency.

10. What is dependency injection in Angular?Dependency injection (DI), is an important application design pattern in which a class asks for dependencies from external sources rather than creating them itself. Angular comes with its own dependency injection framework for resolving dependencies( services or objects that a class needs to perform its function).So you can have your services depend on other services throughout your application.

11. How is Dependency Hierarchy formed?
	Injectors in Angular have rules that can be leveraged to achieve the desired visibility of injectables in your applications. By understanding these rules, you can determine in which NgModule, Component, or Directive you should declare a provider.
	### Angular has two injector hierarchies:
![injector hierarchies](/Attachments/injector%20hierarchies.png)
    ### Module injector
    When angular starts, it creates a root injector where the services will be registered, these are provided via injectable annotation. All services provided in the `ng-model` property are called providers (if those modules are not lazy-loaded).
    Angular recursively goes through all models which are being used in the application and creates instances for provided services in the root injector. If you provide some service in an eagerly-loaded model, the service will be added to the root injector, which makes it available across the whole application.
    ### Platform Module
    During application bootstrapping angular creates a few more injectors, above the root injector goes the platform injector, this one is created by the platform browser dynamic function inside the `main.ts` file, and it provides some platform-specific features like `DomSanitizer`.
    ### NullInjector()
    At the very top, the next parent injector in the hierarchy is the `NullInjector()`.The responsibility of this injector is to throw the error if something tries to find dependencies there, unless you've used `@Optional()` because ultimately, everything ends at the `NullInjector()` and it returns an error or, in the case of `@Optional()`, `null`.
![hierarchy diagram](/Attachments/hierarchy%20diagram.png)
    ### ElementInjector
    Angular creates `ElementInjector` hierarchies implicitly for each DOM element. `ElementInjector` injector is being created for any tag that matches the angular component, or any tag on which directive is applied, and you can configure it in component and directive annotations inside the provider's property, thus, it creates its own hierarchy likewise the upper one.
![element injector hieracrhy](/Attachments/element%20injector%20hieracrhy.png)


12. What happens if you use script tag inside template?
	Angular recognizes the value as unsafe and automatically sanitizes it, which removes the `script` tag but keeps safe content such as the text content of the `script` tag. This way it eliminates the risk of script injection attacks. If you still use it then it will be ignored and a warning appears in the browser console.
	Let's take an example of innerHtml property binding which causes XSS vulnerability,
```plain text
export class InnerHtmlBindingComponent {
  // For example, a user/attacker-controlled value from a URL.
  htmlSnippet = 'Template <script>alert("0wned")</script> <b>Syntax</b>';
}
```

13. What are pipes?
	Pipes are simple functions that use [template expressions](https://github.com/sudheerj/angular-interview-questions#what-are-template-expressions) to accept data as input and transform it into a desired output. For example, let us take a pipe to transform a component's birthday property into a human-friendly date using **date** pipe.
```plain text
import { Component } from '@angular/core';

@Component({
  selector: 'app-birthday',
  template: `<p>Birthday is {{ birthday | date }}</p>`
})
export class BirthdayComponent {
  birthday = new Date(1987, 6, 18); // June 18, 1987
}
```

14. How do you chain pipes?
	You can chain pipes together in potentially useful combinations as per the needs. Let's take a birthday property which uses date pipe(along with parameter) and uppercase pipes as below
```plain text
import { Component } from '@angular/core';

        @Component({
          selector: 'app-birthday',
          template: `<p>Birthday is {{  birthday | date:'fullDate' | uppercase}} </p>` // THURSDAY, JUNE 18, 1987
        })
        export class BirthdayComponent {
          birthday = new Date(1987, 6, 18);
        }
```

15. What is the difference between pure and impure pipe?A pure pipe is only called when Angular detects a change in the value or the parameters passed to a pipe. For example, any changes to a primitive input value (String, Number, Boolean, Symbol) or a changed object reference (Date, Array, Function, Object). An impure pipe is called for every change detection cycle no matter whether the value or parameters changes. i.e, An impure pipe is called often, as often as every keystroke or mouse-move.

16. What is a bootstrapping module?
	Every application has at least one Angular module, the root module that you bootstrap to launch the application is called as bootstrapping module. It is commonly known as `AppModule`. The default structure of `AppModule` generated by AngularCLI would be as follows:
```plain text
    import { BrowserModule } from '@angular/platform-browser';
    import { NgModule } from '@angular/core';
    import { FormsModule } from '@angular/forms';
    import { HttpClientModule } from '@angular/common/http';

    import { AppComponent } from './app.component';

    /* the AppModule class with the @NgModule decorator */
    @NgModule({
      declarations: [
        AppComponent
      ],
      imports: [
        BrowserModule,
        FormsModule,
        HttpClientModule
      ],
      providers: [],
      bootstrap: [AppComponent]
    })
    export class AppModule { }
```

17. What is RxJS?
	RxJS is a library for composing asynchronous and callback-based code in a functional, reactive style using Observables. Many APIs such as HttpClient produce and consume RxJS Observables and also uses operators for processing observables.
	For example, you can import observables and operators for using HttpClient as below,
```plain text
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';
```

18. What is the difference between promise and observable?
	Below are the list of differences between promise and observable:

| Observable | Promise |
| --- | --- |
| Declarative: Computation does not start until subscription, so they can run whenever you need the result | Executes immediately on creation |
| Provides multiple values over time | Provides only one |
| Subscribe method is used for error handling that facilitates centralized and predictable error handling | Push errors to the child promises |
| Provides chaining and subscription to handle complex applications | Uses only `.then()` clause |
19. What is multicasting?
	Multi-casting is the practice of broadcasting to a list of multiple subscribers in a single execution.
	Let's demonstrate the multi-casting feature:
```plain text
var source = Rx.Observable.from([1, 2, 3]);
var subject = new Rx.Subject();
var multicasted = source.multicast(subject);

// These are, under the hood, `subject.subscribe({...})`:
multicasted.subscribe({
  next: (v) => console.log('observerA: ' + v)
});
multicasted.subscribe({
  next: (v) => console.log('observerB: ' + v)
});

// This is, under the hood, `s
```

20. What are Angular elements?Angular elements are Angular components packaged as **custom elements** (a web standard for defining new HTML elements in a framework-agnostic way). Angular Elements host an Angular component, providing a bridge between the data and the logic defined in the component and the standard DOM APIs, thus, providing a way to use Angular components in `non-Angular environments`.

21. What is the browser support of Angular Elements?
	Since Angular elements are packaged as custom elements the browser support of angular elements is same as custom elements support.
	This feature is is currently supported natively in a number of browsers and pending for other browsers.

22. What are custom elements?Custom elements (or Web Components) are a Web Platform feature which extends HTML by allowing you to define a tag whose content is created and controlled by JavaScript code. The browser maintains a `CustomElementRegistry` of defined custom elements, which maps an instantiable JavaScript class to an HTML tag. Currently this feature is supported by Chrome, Firefox, Opera, and Safari, and available in other browsers through polyfills.

23. Do I need to bootstrap custom elements?No, custom elements bootstrap (or start) automatically when they are added to the DOM, and are automatically destroyed when removed from the DOM. Once a custom element is added to the DOM for any page, it looks and behaves like any other HTML element, and does not require any special knowledge of Angular.

24. Explain how custom elements works internally?
	Below are the steps in an order about custom elements functionality,
	1. **App registers custom element with browser:** Use the `createCustomElement()` function to convert a component into a class that can be registered with the browser as a custom element.
	2. **App adds custom element to DOM:** Add custom element just like a built-in HTML element directly into the DOM.
	3. **Browser instantiate component based class:** Browser creates an instance of the registered class and adds it to the DOM.
	4. **Instance provides content with data binding and change detection:** The content with in template is rendered using the component and DOM data. The flow chart of the custom elements functionality would be as follows,
![customElement](/Attachments/customElement.png)
25. How to transfer components to custom elements?
	Transforming components to custom elements involves **two** major steps,
	5. **Build custom element class:** Angular provides the `createCustomElement()` function for converting an Angular component (along with its dependencies) to a custom element. The conversion process implements `NgElementConstructor` interface, and creates a constructor class which is used to produce a self-bootstrapping instance of Angular component.
	6. **Register element class with browser:** It uses `customElements.define()` JS function, to register the configured constructor and its associated custom-element tag with the browser's `CustomElementRegistry`. When the browser encounters the tag for the registered element, it uses the constructor to create a custom-element instance.
	The detailed structure would be as follows,
![createElement](/Attachments/createElement.png)
26. What are the mapping rules between Angular component and custom element?The Component properties and logic maps directly into HTML attributes and the browser's event system. Let us describe them in two steps,
	7. The createCustomElement() API parses the component input properties with corresponding attributes for the custom element. For example, component @Input('myInputProp') converted as custom element attribute `my-input-prop`.
	8. The Component outputs are dispatched as HTML Custom Events, with the name of the custom event matching the output name. For example, component @Output() valueChanged = new EventEmitter() converted as custom element with dispatch event as "valueChanged".
27. How do you define typings for custom elements?
	You can use the `NgElement` and `WithProperties` types exported from @angular/elements.
	Let's see how it can be applied by comparing with Angular component.
	9. The simple container with input property would be as below,
```typescript
@Component(...)
class MyContainer {
  @Input() message: string;
}
```
	10. After applying types typescript validates input value and their types,
```plain text
const container = document.createElement('my-container') as NgElement & WithProperties<{message: string}>;
container.message = 'Welcome to Angular elements!';
container.message = true;  // <-- ERROR: TypeScript knows this should be a string.
container.greet = 'News';  // <-- ERROR: TypeScript knows there is no `greet` property on `container`.

```
28. What are dynamic components? Dynamic components are the components in which the component's location in the application is not defined at build time i.e. they are not used in any angular template. Instead, the component is instantiated and placed in the application at runtime.

29. What is router state?
	RouterState is a tree of activated routes. Every node in this tree knows about the "consumed" URL segments, the extracted parameters, and the resolved data. You can access the current RouterState from anywhere in the application using the `Router service` and the `routerState` property.
```plain text
@Component({templateUrl:'template.html'})
class MyComponent {
  constructor(router: Router) {
    const state: RouterState = router.routerState;
    const root: ActivatedRoute = state.root;
    const child = root.firstChild;
    const id: Observable<string> = child.params.map(p => p.id);
    //...
  }
}
```

30. What are router events?
	During each navigation, the Router emits navigation events through the Router.events property allowing you to track the lifecycle of the route.
	The sequence of router events is as below,
	1. NavigationStart,
	2. RouteConfigLoadStart,
	3. RouteConfigLoadEnd,
	4. RoutesRecognized,
	5. GuardsCheckStart,
	6. ChildActivationStart,
	7. ActivationStart,
	8. GuardsCheckEnd,
	9. ResolveStart,
	10. ResolveEnd,
	11. ActivationEnd
	12. ChildActivationEnd
	13. NavigationEnd,
	14. NavigationCancel,
	15. NavigationError
	16. Scroll

31. What is a service worker and its role in Angular?A service worker is a script that runs in the web browser and manages caching for an application. Starting from 5.0.0 version, Angular ships with a service worker implementation. Angular service worker is designed to optimize the end user experience of using an application over a slow or unreliable network connection, while also minimizing the risks of serving outdated content.
32. What are the design goals of service workers?
	Below are the list of design goals of Angular's service workers,
	17. It caches an application just like installing a native application
	18. A running application continues to run with the same version of all files without any incompatible files
	19. When you refresh the application, it loads the latest fully cached version
	20. When changes are published then it immediately updates in the background
	21. Service workers saves the bandwidth by downloading the resources only when they changed.

33. What is Angular Ivy?
	Angular Ivy is a new rendering engine for Angular. You can choose to opt in a preview version of Ivy from Angular version 8.
	22. You can enable ivy in a new project by using the --enable-ivy flag with the ng new command
```typescript
ng new ivy-demo-app --enable-ivy
```
	23. You can add it to an existing project by adding `enableIvy` option in the `angularCompilerOptions` in your project's `tsconfig.app.json`.
```plain text
{
  "compilerOptions": { ... },
  "angularCompilerOptions": {
    "enableIvy": true
  }
}
```

34. What are the features included in ivy preview?
	You can expect below features with Ivy preview,
	1. Generated code that is easier to read and debug at runtime
	2. Faster re-build time
	3. Improved payload size
	4. Improved template type checking

35. What is Angular DSL?
	A domain-specific language (DSL) is a computer language specialized to a particular application domain. Angular has its own Domain Specific Language (DSL) which allows us to write Angular specific html-like syntax on top of normal html. It has its own compiler that compiles this syntax to html that the browser can understand. This DSL is defined in NgModules such as animations, forms, and routing and navigation.
	Basically you will see 3 main syntax in Angular DSL.
	1. `()`: Used for Output and DOM events.
	2. `[]`: Used for Input and specific DOM element attributes.
	3. : Structural directives(*ngFor or *ngIf) will affect/change the DOM structure.

### What is the purpose of differential loading in CLI?

From Angular8 release onwards, the applications are built using differential loading strategy from CLI to build two separate bundles as part of your deployed application.

36. The first build contains ES2015 syntax which takes the advantage of built-in support in modern browsers, ships less polyfills, and results in a smaller bundle size.
37. The second build contains old ES5 syntax to support older browsers with all necessary polyfills. But this results in a larger bundle size.

**Note:** This strategy is used to support multiple browsers but it only load the code that the browser needs.

### What is lazy loading?

Lazy loading is one of the most useful concepts of Angular Routing. It helps us to download the web pages in chunks instead of downloading everything in a big bundle. It is used for lazy loading by asynchronously loading the feature module for routing whenever required using the property `loadChildren`. Let's load both `Customer` and `Order` feature modules lazily as below,

```plain text
const routes: Routes = [
  {
    path: 'customers',
    loadChildren: () => import('./customers/customers.module').then(module => module.CustomersModule)
  },
  {
    path: 'orders',
    loadChildren: () => import('./orders/orders.module').then(module => module.OrdersModule)
  },
  {
    path: '',
    redirectTo: '',
    pathMatch: 'full'
  }
];
```

### What is Angular security model for preventing XSS attacks?

Angular treats all values as untrusted by default. i.e, Angular sanitizes and escapes untrusted values When a value is inserted into the DOM from a template, via property, attribute, style, class binding, or interpolation.

### What are Http Interceptors?

Http Interceptors are part of @angular/common/http, which inspect and transform HTTP requests from your application to the server and vice-versa on HTTP responses. These interceptors can perform a variety of implicit tasks, from authentication to logging.

The syntax of HttpInterceptor interface looks like as below,

```typescript
interface HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>>
}
```

You can use interceptors by declaring a service class that implements the intercept() method of the HttpInterceptor interface.

```typescript
@Injectable()
 export class MyInterceptor implements HttpInterceptor {
    constructor() {}
    intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
        ...
    }
}
```

After that you can use it in your module,

```plain text
@NgModule({
    ...
    providers: [
        {
            provide: HTTP_INTERCEPTORS,
            useClass: MyInterceptor,
            multi: true
        }
    ]
    ...
})
export class AppModule {}
```


### What is TestBed?

TestBed is an api for writing unit tests for Angular applications and it's libraries. Even though We still write our tests in Jasmine and run using Karma, this API provides an easier way to create components, handle injection, test asynchronous behaviour and interact with our application.

### What is protractor?

Protractor is an end-to-end test framework for Angular and AngularJS applications. It runs tests against your application running in a real browser, interacting with it as a user would.

```plain text
npm install -g protractor
```

### What is collection?

Collection is a set of related schematics collected in an npm package. For example, `@schematics/angular` collection is used in Angular CLI to apply transforms to a web-app project. You can create your own schematic collection for customizing angular projects.

### What is the purpose of hidden property?

The hidden property is used to show or hide the associated DOM element, based on an expression. It can be compared close to `ng-show` directive in AngularJS. Let's say you want to show user name based on the availability of user using `hidden` property.

```plain text
<div [hidden]="!user.name">
  My name is: {{user.name}}
</div>
```