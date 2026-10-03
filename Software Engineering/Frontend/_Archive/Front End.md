[[JS Concepts]]

## Topics

> [!note]+ JavaScript
> To visualize Hoisting [https://ui.dev/javascript-visualizer](https://ui.dev/javascript-visualizer)
> 
> > [!note]+ Scope
> > **Scope** determines where variables are accessible. In JavaScript, there are global scope, module scope (for ES modules), function (local) scope, and block scope
> >     - A **global variable** (declared outside any function or block) is accessible anywhere in the program [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Grammar_and_types#variable_hoisting#:~:text=When%20you%20declare%20a%20variable,available%20only%20within%20that%20function).
> >     - A **function-scoped** variable (declared with `var` inside a function) is only visible within that function (and its nested functions).
> >     - Variables declared with `let` or `const` are **block-scoped**, meaning they exist only within the nearest enclosing `{ ... }` block (such as inside an `if` or `for` block) [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Grammar_and_types#variable_hoisting#:~:text=When%20you%20declare%20a%20variable,available%20only%20within%20that%20function). They are not accessible outside that block. (`var` does *not* have block scope, which often surprises newcomers [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Grammar_and_types#variable_hoisting#:~:text=console,not%20defined).)
> >     - Module scope refers to variables defined in an ES6 module (they don’t pollute global scope).
> > 
> > > TL;DR: It is the region of code where a variable or function is accessible
> > 
> > > TL;DR 2: Where the variable is available to use.
> > 
> > > Types: (Global, Local (Function), Block, Module, Lexical Scope)
> > 
> > Closures “Backpack” is when a function <u>remembers </u>and keep access to variables from its outer scope, even after the outer function has finished running.
> > 
> > The main idea of closures is to keep the lexical environment alive.
> > 
> > JS variables are lexically scoped (We can determine the scope of it from where it is declared), but var is not lexically scoped.
> > The lexical scope is defined like this in the spec:
> > 
> > ```bash
> > LexicalEnvironment = {
> > 	environmentRecord: { a: 1, foo: <Function>, ... },
> > 	outer: <LexicalEnvironment or null>
> > }
> > ```
> > 
> > The `**environmentRecord**`** **is map of identifiers (variable names) to bindings (actual memory locations or values, it includes both declarations (let, const, var, functions) and sometimes mutable state.
> > 
> > `**outer**` lexical environment, is a pointer to the next outer lexical environment (parent lexical scope)
> > 
> > When you debug that in chrome, you will see graphical representation of the scope chain (Lexical Environment Chain)
> > 
> > ![[image 6.png]]
> > 
> > To break them down:
> > 
> > 1. Local, is the lexical environment of the currently executing function, even tho we didn’t declare any variables the JS engine will always create one for each function invocation
> > and here, `this` is set to Window because it was called in the global context, if strict mode was on `this` will be undefined.
> > 2. Closure (outer), the lexical environment captured by the closure, the parent scope.
> > When outer() was called, it created its own lexical environment with a = 2, then it defined inner(), when inner was returned, it also retains a reference to outer lexical environment.
> > 3. Script, This is the top level lexical environment for the script, it includes a: 1, fn = outer() “inner” function “This is sometimes called the Module scope”.
> > 4. Global, this is the `window` object in the browser or the globalThis, it contains all built ins like console, setTimeout, Arrays…etc and any global variable declared with var (not let or const)
> > 
> > JS allows nested scopes, which will create a scope tree (Scope chain).
> > 
> > Each module has its own scope, variables declared at a module level are only available within that module
> > 
> > Closure memory model
> >     This is how JS manages memory for variables that are captured by a function especially after the outer function has finished execution, this includes where these closed over variables are stored, how they are retained in memory and when they are garbage collected.
> > > Closed over variables are stored in **heap**, **Stack **is only used for<u> temporary call frames</u> (execution contexts that go away after return).
> > So when you create a closure, the lexical environment is moved to the heap, and the inner function holds a “hidden” reference to that environment. 
> > 
> >     Let's model memory step by step:
> >     ### 🧱 Stack Frame for `outer()`:
> >     - Holds:
> >         - `count = 0` (initially in the stack)
> >         - Function object `increment`
> >     BUT:
> >     As soon as `increment()` is returned and **referenced outside**, the engine:
> >     1. **Moves the Lexical Environment** containing `count` into the **heap**
> >     2. `increment` gets a **hidden property** like `[[Environment]]` that points to this Lexical Environment
> >     3. That environment is **kept alive** as long as `increment` exists
> > ---
> > ```plain text
> > Heap:
> >   LexicalEnvironment_1 = {
> >     count: 0
> >   }
> > 
> >   increment = {
> >     [[FunctionCode]]: ...
> >     [[Environment]]: LexicalEnvironment_1
> >   }
> > 
> > Stack:
> >   counter = reference to increment (on heap)
> > ```
> >     When you call `counter()`, it:
> >     - Executes function code for `increment`
> >     - Resolves `count` by walking its `[[Environment]]` pointer
> >     - Increments and returns it
> >     The V8 and Node.js don’t store Lexical environments naively, they use hidden classes and context objects, each closure has a `[[environment]]` internal slot pointing to a context, the context is an optimized structure for variable lookup, like a lightweight dictionary. the engine reuse contexts or even eliminate unused bindings if proven to be unused. 
> >     In V8, closures are linked to their environment using an internal structure called <u>**context object**</u> (closure context) which is a lexical environment record that is stored in the heap.
> > > [!note]+ Why are closed over variables stored in the heap, not stack?
> > > Heaps are long-term memory, stack is temporary scratchpad for execution.
> > > 
> > > - The stack is used for ephemeral execution context “Short lived”.
> > > - When a function returns, the stack frame is popped off.
> > > - The variable moved up to heap can live until the closure dies.
> > 
> > 
> 
> > [!note]+ Hoisting 
> > [https://www.scaler.com/topics/hoisting-in-javascript/](https://www.scaler.com/topics/hoisting-in-javascript/)
> > 
> > Hoisting is the JS engine process of moving variable and function declarations to the top of their scope before the code runs. 
> > 
> > A mechanism in JS (Just like Header files in C++) where the declarations are moved to the top of their scope before code execution.
> > 
> > The reason you get a reference error when trying to access `let` or `const` variable before declarations is because of <u>Temporal dead zone</u>.
> > 
> > The **TMZ** starts at the beginning of the variables enclosing scope and ends when it is declared
> > 
> > ```typescript
> > {
> > 	// Start of name TDZ
> > 	let age = 16;
> > 	console.log(age); // 16
> > 	console.log(name); // ReferenceError because we are on the TDZ
> > 	let name = 'Ali'; // End of name TDZ
> > }
> > ```
> > 
> > Function hoisting allows us to call the functions before it is defined, note that only function declarations are hoisted, not function expressions.
> > 
> > Classes are hoisted too, they are undefined in the stage 1 “Creation” of hoisting.
> > 
> > > You can visualize hoisting here: [JavaScript Visualizer](https://ui.dev/javascript-visualizer)
> > 
> > Hoisting have 2 phases 
> >     - Creation phase we do the declaration (Assigning the label)
> >     - Initialization to the default value of the var.
> > > **Creation & Execution**
> > 
> > The **var **lifecycle
> >     In the execution phase we get the value and save it in memory.  
> > 
> > ![[Untitled 44.png]]
> > 
> > Let’s look at another example
> > 
> > ```typescript
> > function add(x,y) {        //function declaration
> >   var result = x + y
> >   return result
> > }
> > console.log(add(2,3))
> > ```
> > 
> > The **creation **phase
> >     - The JavaScript engine <u>scans</u> the code during the creation phase.
> >     - It recognizes the add function declaration and allocates memory for it.
> >     - The console.log statement is also noted.
> > 
> > The **execution **phase
> >     - The execution starts, and when the console.log is encountered, it sees the function add.
> >     - Although the function is called before its declaration, the hoisting mechanism moves the function to the top during the creation phase.
> >     - The function is available, and the output is 5, the result of add(2, 3).
> > 
> 
> > [!note]+ JS Call Stack, Event Loops 
> > ![[Screenshot_2023-08-08_at_3.44.28_PM.png]]
> > 
> > ![[Screenshot_2023-08-08_at_3.26.42_PM.png]]
> > 
> > ![[Screenshot_2023-08-08_at_3.27.09_PM.png]]
> > 
> > Priority
> >     1. Call Stack → Execute current stack of synchronous code
> >     2. Micro Tasks → After the call stack is cleared and after each macrotask, all available microtasks are executed. If a microtask adds another microtask it will be executed before moving to the next phase. “<u>The rendering happens here</u>”
> >     3. Macro Tasks → After all available microtasks are executed and event loop will start processing the macrotasks queue. 
> > 
> > Call stack
> > ![[Screenshot_2023-08-08_at_3.27.27_PM.png]]
> >     The call stack is mechanism that JS Engines (like v8) uses to keep track of where the thread is in a script or a program, it is a **Stack data** (LIFO Last in first out) structure where function calls are added to and removed from the top.
> >     Every time a function is called, it is added to the top of the stack, when the function finishes its execution or returns it is removed from the top.
> >     <u>JS has a single call stack.</u>
> > 
> > Web API
> > ![[Screenshot_2023-08-08_at_3.27.56_PM.png]]
> >     When JS runs code that requires background tasks like (timers, AJAX requests or events) it uses the WebAPI provided by the browser environment. These are functionalities not provided by the JS language but by the browser.
> >     When you execute `setTimeOut`, JS will use the browser web API to handle the timer, freeing up the call stack to process other things.
> > 
> > Callback Queue (Task Queue)
> >     Once the browser finish a task (timer, ajax request finishes) it puts its corresponding callback function into a task queue or a callback queue.
> > 
> > Event Loop
> > ![[Screenshot_2023-08-08_at_3.28.59_PM.png]]
> >     It’s main primary role is to look at the call stack and the task queue, if the call stack is empty. it takes the first event from the queue and pushes it onto the stack, starting its execution.
> > 
> > MacroTasks
> > ![[Screenshot_2023-08-08_at_3.28.18_PM.png]]
> >     The standard tasks in the callback queue, like setTImeOut setInterval and UI rendering tasks.
> >     After the event loop process all the microtasks it will start with macro tasks.
> >     like setTimeout, setInterval, I/O operations, FetchAPICallbacks
> > 
> > MicroTasks
> > ![[Screenshot_2023-08-08_at_3.28.39_PM.png]]
> >     They are higher priority than MacroTasks, and at every run of the event loop and after every task the **JS** engine checks if there’s any microtask queued, if there are they will all be run one by one until the microtask queue is emptied.
> >     Tasks that should be executed right after the current executing script complete, and before processing any other task in the queue. like Promise.then(), catch, finally or MutatuionObserver.
> > 
> 
> > [!note]+ Execution Context & call, bind, apply
> > Abstract concept that represent the environment where the code is evaluated and executed.
> > 
> > > [!note]+ Main components of the execution context:
> > > Variable environment
> > >     Contains variables, function declarations and arguments to functions. 
> > > 
> > > Scope chain
> > >     Provides ordered access to all variables and function an execution context has access to, also it includes it own variable environment and variable environment of it’s parents (or outer) execution context.
> > > 
> > > this binding(`this`)
> > >     Refers to an object that is the current code execution context.
> > 
> > > [!note]+ Types of execution context
> > > Global execution context
> > >     Created by default, one per program, All global code (Code outside of functions) is executed in the global context, `this`  value refers to the global `window` object in browsers.
> > > 
> > > Function execution context
> > >     Created each time a function is invoked/called. The details of the context depends on how the function is called, which impact the value of `this`.
> > > 
> > > Eval execution context
> > >     It evaluates the code inside and run it. (NEVER EVER USE EVAL)
> > 
> > > [!note]+ bind, call, apply
> > > ```typescript
> > > function setName() {
> > >   this.name = "Mohammad";
> > > }
> > > 
> > > const object = {
> > >   name: "Ali",
> > >   changeName: function () {
> > > 		// try apply, call, and bind
> > > 		// setName.bind(this)(); // This here because we need to invoke bind
> > >     // setName.apply(this);
> > > 		// setName.call(this);
> > >   }
> > > };
> > > 
> > > console.log("Name: ", object.name);
> > > object.changeName();
> > > console.log("Name: ", object.name);
> > > ```
> > > 
> > > the value of `this` is determined based on how the function is called.
> > > 
> > > These methods allow us to explicitly set what you want to refer as `this`
> > > 
> > > call
> > >     Invokes the function with a specified this context and arguments
> > > 
> > > apply
> > >     Similar to call but the function parameters are passed as an array
> > > 
> > > bind
> > >     Returns a new function (Make a new copy) allowing you to bind a function to a specific `this` context, it doesn’t immediately invoke the function.
> 
> > [!note]+ var vs let vs const
> > Var
> >     var scope is global or function/locally scoped, it is globally scoped when it is outside of everything, and is locally scoped when it is inside a function.
> > ```typescript
> > var name = "Ali";
> > 
> > sayHi();
> > 
> > function sayHi() {
> >   // This is locally scoped to the function 
> >   // (New execution context)
> >   var name = "Mohammad";
> >   console.log(name); // Mohammad
> > }
> > 
> > if (true) {
> >   // This will re-define the global name
> >   var name = "Canaan";
> > }
> > 
> > console.log(name); // Canaan
> > ```
> >     Var can be re-declared and updated, this means in the same scope we can do 
> > ```typescript
> > var ali = 'Mohammad'
> > var ali = 'Canaan'
> > 
> > // and also
> > var ali = 'Mohammad'
> > ali = 'Canaan'
> > ```
> >     the hoisting of the var is moved up from it’s declaration, so if you do it like this 
> > ```typescript
> > console.log(ali); // undefined
> > var ali = 'Mohammad';
> > 
> > // it will be evaluated to 
> > var ali;
> > console.log(ali);
> > ali = 'Mohammad';
> > ```
> > 
> > let
> >     Let is an improvement on var, it comes to solve the globalized scope issue of var.
> >     Let is blocked scoped, means it is scoped with {}.
> >     Let can be updated, but not re-declared within its scope
> > ```typescript
> > // This will work
> > let name = 'Ali';
> > name = 'Canaan';
> > 
> > // This will give an error
> > let greeting = 'Hi';
> > let greeting = 'Something'; // This will throw an error
> > 
> > // This is fine, because of the block scope
> > let name = 'Ali';
> > if (true) {
> > 	let name = 'Canaan';
> > 	console.log(name); // Canaan
> > }
> > console.log(name); // Ali
> > ```
> >     Hoisting of let is just like var, it is hoisted to the top, but unlike var it is not initialized with undefined, if you try to access it, it will give you a <u>Reference Error</u>.
> > 
> > Const
> >     variables declared with `const` maintain constant values.
> >     const is blocked scoped, and can’t be updated nor re-declared.
> >     Hoisting of const is just like let, moved up but not initialized.
> > 
> 
> > [!note]+ Closures 
> > [[Closures|Closures]]
> > 
> > A closure is a function combined with reference to the variable defined outside of it, closures maintain references to the variables outside it, they enclose the function.
> > 
> > Closures do not freeze the value of variables from function’s outer scope during creation, they maintain the reference throughout the closures lifetime.
> > 
> > <u>Scope is just really what variable (data) is available for me in that exact line that I’m running</u>.
> > 
> > JS Scope rule is **lexical/static** scoping: Where i save my function determines for the rest of that function life what data it will have access to. (Not where you run it That is *Dynamic Scoping*) so even if you returned the function you will grab the data and save it.
> > 
> > They refer to the “backpack” as (Closed Over Variable Environment C.O.V.E) or (<u>Persisted Lexical/Static Scoped Referenced Data</u>) then they just named it a **Closure**.
> > 
> > The backpack is the result of lexical scoping.
> 
> > [!note]+ Anonymous Functions
> > Anon function are the functions with no name, and intended to be used one time, you can see them with callbacks.
> 
> > [!note]+ Higher order functions & Currying
> > A HoF is a function that takes a one or more parameters and returns a function, a common examples is like filter, reduce, map.
> > 
> > ```typescript
> > const numbers = [1, 2, 3, 4];
> > const double = numbers.map((x) => x * 2); 
> > ```
> > 
> > Currying
> >     a technique of translating the evaluations of a function that takes multiple arguments into a sequence of functions each take an argument with each returning a new function that takes the second argument and third and so on.
> > ```typescript
> > function multiply(a) {
> > 	return function (b) {
> > 		return function (c) {
> > 				return a * b * c;
> > 			}
> > 	}
> > }
> > 
> > const double = muliply(2); // a = 2
> > const twoDouble = double(5); b = 5
> > twoDouble(10); // c = 10
> > ```
> > 
> 
> > [!note]+ Arrays
> > In JavaScript, arrays are just another kind of object. [Like with objects](https://react.dev/learn/updating-objects-in-state), **you should treat arrays in React state as read-only.** This means that you shouldn’t reassign items inside an array like `arr[0] = 'bird'`, and you also shouldn’t use methods that mutate the array, such as `push()` and `pop()`.
> > 
> > Instead, every time you want to update an array, you’ll want to pass a *new* array to your state setting function. To do that, you can create a new array from the original array in your state by calling its non-mutating methods like `filter()` and `map()`. Then you can set your state to the resulting new array.
> > 
> > Here is a reference table of common array operations. When dealing with arrays inside React state, you will need to avoid the methods in the left column, and instead prefer the methods in the right column:
> > 
> > |   | avoid (mutates the array) | prefer (returns a new array) |
> > | --- | --- | --- |
> > | adding | `push`, `unshift` | `concat`, `[...arr]` spread syntax ([example](https://react.dev/learn/updating-arrays-in-state#adding-to-an-array)) |
> > | removing | `pop`, `shift`, `splice` | `filter`, `slice` ([example](https://react.dev/learn/updating-arrays-in-state#removing-from-an-array)) |
> > | replacing | `splice`, `arr[i] = ...` assignment | `map` ([example](https://react.dev/learn/updating-arrays-in-state#replacing-items-in-an-array)) |
> > | sorting | `reverse`, `sort` | copy the array first ([example](https://react.dev/learn/updating-arrays-in-state#making-other-changes-to-an-array)) |
> > 
> > Alternatively, you can [use Immer](https://react.dev/learn/updating-arrays-in-state#write-concise-update-logic-with-immer) which lets you use methods from both columns.
> > 
> > - `slice` lets you copy an array or a part of it.
> > - `splice` **mutates** the array (to insert or delete items).
> 
> > [!note]+ Objects
> > An object like this appears “nested” in code:
> > 
> > ```sql
> > let obj = {
> >   name: 'Niki de Saint Phalle',
> >   artwork: {
> >     title: 'Blue Nana',
> >     city: 'Hamburg',
> >     image: 'https://i.imgur.com/Sd1AgUOm.jpg',
> >   }
> > };
> > ```
> > 
> > However, “nesting” is an inaccurate way to think about how objects behave. When the code executes, there is no such thing as a “nested” object. You are really looking at two different objects:
> > 
> > ```sql
> > let obj1 = {
> >   title: 'Blue Nana',
> >   city: 'Hamburg',
> >   image: 'https://i.imgur.com/Sd1AgUOm.jpg',
> > };
> > 
> > let obj2 = {
> >   name: 'Niki de Saint Phalle',
> >   artwork: obj1
> > };
> > ```
> > 
> > The `obj1` object is not “inside” `obj2`. For example, `obj3` could “point” at `obj1` too:
> > 
> > ```sql
> > let obj1 = {
> >   title: 'Blue Nana',
> >   city: 'Hamburg',
> >   image: 'https://i.imgur.com/Sd1AgUOm.jpg',
> > };
> > 
> > let obj2 = {
> >   name: 'Niki de Saint Phalle',
> >   artwork: obj1
> > };
> > 
> > let obj3 = {
> >   name: 'Copycat',
> >   artwork: obj1
> > };
> > ```
> > 
> > If you were to mutate `obj3.artwork.city`, it would affect both `obj2.artwork.city` and `obj1.city`. This is because `obj3.artwork`, `obj2.artwork`, and `obj1` are the same object. This is difficult to see when you think of objects as “nested”. Instead, they are separate objects “pointing” at each other with properties.
> > 
> > In `Immer`  If you want to update deeply nested object use an `draft` which is an object called a [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy), that “records” what you do with it. This is why you can mutate it freely as much as you like! Under the hood, Immer figures out which parts of the `draft` have been changed, and produces a completely new object that contains your edits.
> 
> > [!note]+ Event Propagation (Bubbling)
> > Event handlers will also catch events from any children your component might have, we say an event “bubbles” or “propagates” up the tree. it starts where the event happened and goes up from there.
> > 
> > All events in react will propagate except onScroll.
> > 
> > **Stop Propagation** 
> > 
> > If you want to prevent an event from reaching parent components you can call
> > 
> > ```sql
> > e.stopPropagation();
> > ```
> > 
> > You can capture phase events (Even when you stop propagation with onClickCapture)
> > 
> > Prevent Default
> >     Some browser events have default behaviors attached to them, you can call 
> > ```sql
> > e.preventDefault()
> > ```
> >     on the event object to stop that default behavior.
> > 
> > Prevent Default & Propagation
> > 
> > - [`e.stopPropagation()`](https://developer.mozilla.org/docs/Web/API/Event/stopPropagation) stops the event handlers attached to the tags above from firing.
> > - [`e.preventDefault()`](https://developer.mozilla.org/docs/Web/API/Event/preventDefault)[ ](https://developer.mozilla.org/docs/Web/API/Event/preventDefault)prevents the default browser behavior for the few events that have it.
> 
> > [!note]+ Web Workers & Service Workers
> > > [!note]+ Web Workers
> > > It is a JS Browser API that allows us to run scripts in the background thread, it can perform tasks without interfering with the user interface. In addition, they can preform I/O using the `XMLHTTPRequest` or `fetch`. A worker can send messages to the JS code that created it by posting messages to an event handler specified by that code (And vice versa).
> > > 
> > > In react add the `comlink` and `woker-plugin`
> > > 
> > > There’s two types of running your code
> > >     Sync
> > >         Every time our event loop gets to the task it will execute it until the queue is empty
> > >     Async
> > >         The code will run in concurrency, the engine will jump from one queue to another until they are finished  
> > > 
> > > A <u>**worker**</u> is an object created that runs a named JS file, this file contains the code that will run the worker thread. Workers run in another global context that is different from the current window, thus, using window to get the global scope (instead of self) will through an error.
> > > 
> > > The worker context is represented by `DedicatedWorkerGlobalScope` Object and Shared Workers `SharedWorkerGlobalScope`.
> > > 
> > > A dedicated worker is only accessible from the script that first spawned it. whereas shared workers can be accessed from multiple scripts. 
> > > 
> > 
> > > [!note]+ Service Workers 
> > > They are based on the web workers, they are related to the caching, they act as a proxy between the request and the app. 
> 
> > [!note]+ Shallow Vs Deep Copy
> > Shallow Copy
> >     It creates <u>**a new object or array**</u> that hold *references *to the original object’s properties or array elements, changes made to the nested objects will be reflected in both the original and the copied objects.
> >     To create a shallow copy
> >     Arrays
> >         Use the `slice()` method
> >         Use the `spread operator` (it only copies things one level deep)
> >     Objects
> >         use `Object.assign()`
> >         use the spread operator (it only copies things one level deep)
> > 
> > Deep Copy
> >     It means <u>**creating a new object or an array and recursively copying**</u> all the original object properties or the original array’s elements. 
> >     JS Doesn’t provide a deep copy function but you can create a deep copy with
> >     `JSON.parse(JSON.stringify(orignalObject));`
> >     or using loadash `cloneDeep()`
> > 
> 
> > [!note]+ Promises
> > ![[Screenshot_2023-08-08_at_4.04.04_PM.png]]
> > 
> > A promise is an object that represent an eventual completion or failure of an async operation and its resulting value.
> > 
> > When you create a promise, it takes an executer function as it’s argument, the code there is run sync until a await or resolve, reject is executed.
> > 
> > Promise status:
> >     `Pending` → Init state
> >     `Fulfilled` → Operation is completed successfully
> >     `Rejected` → The Operation failed
> > 
> > Util functions
> >     `Promise.all`
> >         Waits <u>**for all promises**</u> in an iteratable manner to be fulfilled, if `*one *`*get rejected the resulting promise is rejected*.
> >     `Promise.race`
> >         Returns a promise that <u>**fulfills or rejects as soon as **</u><u>`**one **`</u><u>**of the promises**</u> in the iteratable fulfills or rejects.
> >     `Promise.allSettled`
> >         it doesn’t <u>**reject if one of the promises is rejects, instead it waits for all promises to be settled and returns a promise that always fulfills**</u> with an array of objects.
> 
> 
> [[Principles|Principles]]
> 
> [[Functions & Callbacks|Functions & Callbacks]]
> 
> [[Asynchronous JavaScript|Asynchronous JavaScript]]
> 
> 

<!-- Column 1 -->
> [!note]+ React
> > [!note]+ Internal React
> > React uses the **tree structures** to manage and model the UI, Browsers use this structure to model the UI, the DOM represent the HMTL elements and the **CSSOM **does the same for CSS, also there’s **Accessibility Tree**.
> > 
> > ![[Untitled 45.png]]
> > 
> > <u>The state lives inside React</u>, React associates each piece of state it’s holding with the correct component by where the component sits in the UI tree.
> > 
> > `Index: ComponentData` 
> > 
> > <u>Each component</u> in the Screen **has **<u>fully isolated state</u>
> > 
> > > [!note]+ How react renders 
> > > 1. Triggering a render
> > > • **During the initial render,** React will [create the DOM nodes](https://developer.mozilla.org/docs/Web/API/Document/createElement) for `<section>`, `<h1>`, and three `<img>` tags.
> > > There are 2 <u>reasons</u> why a component render
> > > <u>1. It is the component initial render
> > > </u>You need to trigger the initial render, this is done by calling `createRoot` and then calling its `render` function.
> > > 2. <u>The component or the component parent(s) state has been updated</u>
> > > **Updating your state automatically **<u>**queues**</u>** a render.**
> > > 2. Rendering the component 
> > > Rendering means that react is calling your component as a <u>function</u>. The JSX returned from that function is like a <u>**snapshot**</u> of the UI in time.
> > > React here calls the function component whose state update triggered the render.
> > > This process is <u>recursive</u>
> > > • **During a re-render,** React will calculate which of their *properties*, if any, have changed since the previous render. It won’t do anything with that information until the next step, the commit phase.
> > > When react re-renders a component: 
> > >     1. React calls your function again.
> > > 2. Your function returns a new JSX snapshot.
> > > 3. React then updates the screen to match the snapshot you’ve returned.
> > > The state actually lives inside react (Global)
> > > 3. Committing to the DOM
> > > After rendering (calling) your components, react will modify the DOM. 
> > > For the initial render, React will use `appendChild` (DOM API) to put all the DOM nodes it has created on the screen.
> > > For re-renders, React will apply the <u>minimal necessary operations</u> to make the DOM match the latest rendering output.
> > > 
> > > After that, the browser will **repaint** the screen.
> > 
> > > [!note]+ Virtual Dom
> > > It’s a pattern rather than a technology.
> > > 
> > > Is a concept where a <u>virtual representation of a UI is kept in memory and synced with the “Real” DOM </u>by libraries like ReactDOM, the syncing process is known as <u>**reconciliation**</u>. 
> > > 
> > > This approach enables React to make sure the DOM matches the state, this abstracts out the attribute manipulation, event handling and manual DOM updating you would do otherwise. 
> > > 
> > > Virtual DOM in react is usually associated with React elements since they are the objects that represents the user interface, also there’s internal objects called “Fibers” to hold additional information about the component tree.
> > 
> > > [!note]+ Shadow DOM
> > > Is a browser technology designed for scoping variables and CSS in web components.
> > 
> > > [!note]+ React Fiber 
> > > is a new reconciliation engine that has a goal to enable <u>incremental rendering of the VirtualDOM</u>
> > 
> > > [!note]+ React KEY (Like the hashmap key)
> > > The key helps react infer what happened in what component to make the correct update to the DOM tree.
> > > 
> > > - **Keys must be unique among siblings.** However, it’s okay to use the same keys for JSX nodes in *different* arrays.
> > > - **Keys must not change** or that defeats their purpose! Don’t generate them while rendering.
> > > 
> > > Similarly, do not generate keys on the fly, e.g. with `key={Math.random()}`. This will cause keys to <u>never match up between renders</u>, leading to all your components and DOM being **recreated** every time. Not only is this slow, but it will also lose any user input inside the list items. Instead, use a stable ID based on the data.
> > > 
> > > Note that your components won’t receive `key` as a prop. It’s only used as a hint by React itself. If your component needs an ID, you have to pass it as a separate prop: `<Profile key={id} userId={id} />`.
> 
> > [!note]+ Typescript
> > Generics
> > 
> > Interfaces vs Abstract Vs Type
> > 
> > Enum Vs Type
> > 
> > Omit
> > 
> > RequiredType
> > 
> 
> In React, **side effects usually belong inside **[**event handlers.**](https://react.dev/learn/responding-to-events) Event handlers are functions that React runs when you perform some action—for example, when you click a button. Even though event handlers are defined *inside* your component, <u>they don’t run </u><u>*during*</u><u> rendering</u>! **So event handlers don’t need to be pure.**
> 
> React have <u>three kinds of inputs that you can read while rendering</u>: [props](https://react.dev/learn/passing-props-to-a-component), [state](https://react.dev/learn/state-a-components-memory), and [context.](https://react.dev/learn/passing-data-deeply-with-context) You should always treat these inputs as <u>**read-only (immutable)**</u>.
> > [!note]+ Context
> > *Context provides a way to pass data through the component tree without having to pass props down manually at every level*
> > 
> > An alternative to passing props, <u>Context lets a parent component provide data to the entire tree below it.</u>
> > 
> > **Just because you need to pass some props several levels deep doesn’t mean you should put that information into context.**
> > 
> > ![[Screenshot_2023-08-01_at_5.13.25_PM.png]]
> > 
> > First create a context with createContext, the only argument to createContext is the default value.
> > 
> > Then use the context, import it and  useContext with the created context
> > 
> > useContext is a hook, just like useState and useReducer, you can call a hook immediately inside a react component.
> > 
> > useContext tell React that the Heading component wants to read the CreatedContext.
> > 
> > Provide the context, and consume it
> > 
> > App level generalized state (Bad Idea)
> > 
> > `**Provider**` allows us to "declare the data that we want available throughout our component tree".
> > 
> > `**Consumer**` allows "any component in the component tree that needs that data to be able to subscribe to it".
> > 
> 
> > [!note]+ State
> > ![[Screenshot_2023-07-31_at_4.14.58_PM.png]]
> > 
> > **React batches state updates, React waits until *****all***** code in the event handlers has run before processing your state updates.**
> > 
> > This lets you update multiple state variables—even from multiple components—without triggering too many [re-renders.](https://react.dev/learn/render-and-commit#re-renders-when-state-updates) But this also means that the UI won’t be updated until *after* your event handler, and any code in it, completes. This behavior, also known as <u>**batching**</u>
> > 
> > It is an uncommon use case, but if you would like to update the same state variable multiple times before the next render, instead of passing the *next state value* like `setNumber(number + 1)`, you can pass a <u>*function*</u> that calculates the next state based on the previous one in the queue, like `setNumber(n => n + 1)`. It is a way to tell React to “<u>do something with the state value</u>” instead of just <u>replacing</u> it.
> > 
> > Here, `n => n + 1` is called an **updater function.** When you pass it to a state setter:
> > 
> > 1. React queues this function to be processed after all the other code in the event handler has run.
> > 2. During the next render, React goes through the queue and gives you the final updated state.
> > 
> > `setNumber(n => n + 1);setNumber(n => n + 1);setNumber(n => n + 1);`
> > 
> > Here’s how React works through these lines of code while executing the event handler:
> > 
> > 3. `setNumber(n => n + 1)`: `n => n + 1` is a function. React adds it to a queue.
> > 4. `setNumber(n => n + 1)`: `n => n + 1` is a function. React adds it to a queue.
> > 5. `setNumber(n => n + 1)`: `n => n + 1` is a function. React adds it to a queue.
> > 
> > When you call `useState` during the next render, React goes through the queue. The previous `number` state was `0`, so that’s what React passes to the first updater function as the `n` argument. Then React takes the return value of your previous updater function and passes it to the next updater as `n`, and so on:
> > 
> > | queued update | `n` | returns |
> > | --- | --- | --- |
> > | `n => n + 1` | `0` | `0 + 1 = 1` |
> > | `n => n + 1` | `1` | `1 + 1 = 2` |
> > | `n => n + 1` | `2` | `2 + 1 = 3` |
> > 
> > React stores `3` as the final result and returns it from `useState`.
> > 
> > This is why clicking “+3” in the above example correctly increments the value by 3.
> > 
> > State behaves like a <u>snapshot</u>, Setting it does not change the state variable you already have, but instead triggers a <u>**re-render**</u>.
> > 
> > data that changes over time is called *state*
> > 
> > Local variables don’t persist between renders, when react renders a component a second time, it renders it from SCRATCH and any change to the local variables won’t trigger renders.
> > 
> > **A state variable’s value never changes within a render,** even if its event handler’s code is asynchronous. Inside *that render’s* `onClick`, the value of `number` continues to be `0` even after `setNumber(number + 5)` was called. Its value was “fixed” when React “took the snapshot” of the UI by calling your component.
> > 
> > To <u>**update a component**</u> two things need to happen
> > 
> > 6. Retain the data between renders
> > 7. Trigger react to render the component with new data (re-rendering)
> > 
> > Objects in React state are technically mutable, you should treat them **as if** they were immutable—like numbers, booleans, and strings. Instead of mutating them, you should always <u>**replace**</u> them.
> > 
> > When you want to *change* something in response to user input, you should [set state](https://react.dev/learn/state-a-components-memory) instead of writing to a variable. You should never change pre-existing variables or objects while your component is rendering.
> 
> > [!note]+ Props
> > <u>Props are the information that you pass to a JSX tag</u>
> > 
> > Props are [immutable](https://en.wikipedia.org/wiki/Immutable_object)—a term from computer science meaning “unchangeable”. When a component needs to change its props (for example, in response to a user interaction or new data), it will have to “ask” its parent component to pass it *different props*—a new object! Its old props will then be cast aside, and eventually the JavaScript engine will reclaim the memory taken by them.
> > 
> > **Don’t try to “change props”.** When you need to respond to the user input (like changing the selected color), you will need to “set state”,
> > 
> > Props reflect a component’s data at any point in time, rather than only in the beginning
> > 
> > One way data flow
> >     It is called unidirectional data flow, one way data flow from the parent to the child to make sure a clean data architecture is followed
> 
> > [!note]+ React Components
> > If we create components inside other components, they will re-mount themselves on every re-render? Or why we don't need the "key" attribute outside of lists. even if we're rendering the same data with the same behavior.
> >     `Fragment`
> >         Fragment lets you group things without leaving any trance in the browser HTML tree, it is the same in the DOM as if the elements where not grouped 
> > ```sql
> > <> Or <Fragment>
> > 
> > </> Or </Fragment>
> > ```
> >     `Profiler`
> >         Let’s you measure rendering performance of React tree programmatically
> >             `id` → The prop id of the Profiler tree that has just committed
> >             `phase` → What is the phase of the tree (mounted, update, nested-update)
> >             `actualDuration` → the number of milliseconds spent render the profile and its descendants for the current update, this indicate how well the subtree used memo and useMemo
> >             `baseDuration` → The number of milliseconds estimating how much time it would take to re-render the entire Profile subtree without any optimization.
> >             `startTime` → When the it began to render the currentUpdate
> >             `endTime` → When it committed the currentUpdate
> > ```sql
> > function onRender(id, phase, actualDuration,
> > 	 baseDuration, startTime, commitTime) {
> >   // Aggregate or log render timings...
> > }
> > 
> > <Profiler id="App" onRender={onRender}>
> > 	<App />
> > </Profiler>
> > ```
> >     `Suspense`
> >         lets you display a fallback until it’s children have finished loading
> > ```sql
> > <Suspense fallback={<Loading />}>
> > 		<App />
> > </Suspens>
> > ```
> >     `StrictMode`
> >         It enables development only behaviour like <u>re-render an extra time, re run effects an extra time, check for usage of deprecated APIs</u>
> > ```sql
> > <StrictMode>
> > 	<App />
> > </StrictMode>
> > ```
> > 
> > ![[Untitled 46.png]]
> > 
> > > [!note]+ React Component Types
> > > Pure
> > >     In the pure components you can change only local variables (Local mutations)
> > >     **React assumes that every component you write is a pure function, as your component will return the same JSX given the same input**
> > >     A pure component mind his own business as it doesn't change any object or variable that existed before it was called, and always have the same input will produce the same output.
> > >     React’s rendering process must always be **pure**. Components should only *return* their JSX, and not *change* any objects or variables that existed before rendering—that would make them impure!
> > > 
> > > Polymorphic Components
> > >     A component that can be rendered in different forms (h1, anchor, …etch) 
> > > 
> > > HOCs (Higher Order Component)
> > >     (Wrapper) A component that takes a component and returns a new component
> > > ```typescript
> > > const Enhanced = higherOrderComponent(WrappedComponet)
> > > ```
> > > 
> 
> > [!note]+ React Hooks 
> > Hooks are ***functions that let you “hook into” React state and lifecycle features from function components***
> > 
> > A** function that lets you interact with the React state and life cycle in function components**
> > 
> > The react hooks lifecycle 
> > 
> > [https://wavez.github.io/react-hooks-lifecycle/](https://wavez.github.io/react-hooks-lifecycle/)
> > 
> > ![[Screenshot_2023-08-01_at_10.38.23_PM.png]]
> > 
> > ![[Untitled 47.png]]
> > 
> > The <u>useLayoutEffect will be fired before the useEffect</u>
> > 
> > ![[Screenshot_2023-08-01_at_10.46.26_PM.png]]
> > 
> > They are a simple functions that we can use to isolate the reusable part from the functional component
> > 
> > Hooks are available while react is <u>rendering</u>.
> > 
> > Hooks are just a special functions that let your component use React features
> > 
> > (setState, useEffect, React Router, useParams)
> > 
> > `useEffect`
> >     The use effect code is run asynchronously **after** React <u>renders</u> the component, the <u>code inside this hook will run after the DOM is painted to the screen</u>.
> >     A hook that let us react to the state changes, fire api calls and much more.
> >     The useEffect has a cleanup function that will be run after the each run of the useEffect.
> > ```typescript
> > // First Run
> > —> useEffectRun
> > 
> > // Second Run
> > —> useEffect Cleanup
> > —> useEffectRun
> > ```
> > 
> > `useState`
> >     It works on the <u>execution context of the component rendering</u>
> >     A hook that let’s you declare a state variable, it has 2 pair of values, the current state and a state setter function that lets you update it.
> >     The state variable to retain the data between renders
> >     A state setter to update the variable and trigger react to render the component again.
> >     Internally, React holds an array of state pairs for every component. It also maintains the current pair index, which is set to `0` before rendering. Each time you call `useState`, React gives you the next state pair and <u>increments the index</u>.
> > 
> > `useTransition`
> >     This hook will let us specify some state updates <u>as not that important</u>, these updates will be executed in **parallel** with other state updates but rendering the component will <u>not wait for these less important state updates</u>.
> >     It will return an array with a variable stating the <u>pending/loading</u> status, and the startTransition function that takes a single **callback** and this callback will contain all the code related to the <u>slow state update including the setting of the state</u>.
> > ```typescript
> > const [name, setName] = useState("");
> > const [list, setList] = useState([]);
> > const [isListTransitionPending, startListTransition] = useTransition();
> > 
> > function handleChange(e) {
> > 	setName(e.target.value)
> > 	startListTransition(() => {
> > 		setList(largeList.filter...);
> > 	});
> > }
> > 
> > return (
> > 	<>
> > 		{ isListTransitionPending ? (<>Loading..</>) : 
> > 				( <List list={list} />)
> > 		}
> > 	</>
> > );
> > ```
> > 
> > `useDifferedValue`
> >     This hook allows us to **fix** the slow <u>render problem</u> (*useMemo with huge calculation*) by implementing a <u>delay</u> before this information is calculated (*Similar to to debouncing and throttling*), **it will only be calculated after all the important state updates have finished running**.
> >     This hook takes a <u>single param</u> which is the value that you want to debounce on, this hook will return a value which will stay the same <u>since this hook will not immediately update the value</u>. This allows time for our component to completely render with the new name value since our list will not try to update immediately while the list will be delayed in its update.
> > ```typescript
> > const [name, setName] = useState("");
> > const defferedName = useDifferedValue(name);
> > 
> > const list = useMemo(() => {
> > 	return largeList.filter...
> > }, [defferedName]);
> > ```
> >     React will be forced to do extra re-renders of your app. This is because it will do the initial re-render and the deferred re-render when it could have done them both in one update without `useDeferredValue`.
> > 
> > `useLayoutEffect`
> >     This will run immediately <u>after</u> the **render** function of the component.
> >     Identical to the useEffect <u>**but**</u> the hook runs the code **synchronously** after React <u>calculate the DOM changes but before it paints those changes to the screen</u>. This means the `useLayoutEffect` will **delay** the <u>painting</u> of the component.
> > 
> > `useRef`
> >     `refs` <u>are just an object for storing a value that persists between renders.</u>
> >     Creates a **reference** for you to work with that are <u>persisted between render</u> and refs <u>does not cause the component to re-render</u> when changed.
> >     Why to use a `ref` ? Imagine that you want to make an input focused, each DOM element have a reference attached to it that you can grab.
> > ```typescript
> > const inputRef = useRef(null);
> > 
> > const focusInput = () => {
> > 	inputRef.current.focus();
> > }
> > 
> > return (
> > 	<>
> > 		<input ref={inputRef} />
> > 		<button onClick={focusInput}>Focus Input </button>
> > 	</>
> > );
> > ```
> >     `React.forwardRef`
> >         A way to reference custom components as if you want to use the custom input Ref 
> > ```typescript
> > const inputRef = useRef(null);
> > 
> > const focusInput = () => {
> > 	inputRef.current.focus();
> > }
> > 
> > return (
> > 	<>
> > 		<CustomInput ref={inputRef} />
> > 		<button onClick={focusInput}>Focus Input </button>
> > 	</>
> > );
> > 
> > 
> > // CustomInput.tsx
> > const CustomInput = (props, ref) => {
> > 	<input ref={ref} />
> > };
> > 
> > export default React.forwardRef(CustomInput);
> > ```
> > 
> > `useImpertiveHandle`
> >     It lets you create a <u>completely custom value</u> for the `ref` you return from your custom component.
> >     It takes two params, the first is the `ref` that you are <u>overriding</u> and the second param is the **function** which returns the new value for your `ref`. and an <u>optional</u> third parameter is the **dependency array** which will make the `impertiveHandle` to re run and update the value of your `ref`. If you didn’t pass the any dependency, the `impertiveHandle` will <u>rerun the function you pass to it and update your ref every time the component is rendered</u>.   
> > 
> > `useMemo`
> >     Prop drilling
> > ![[Screenshot_2023-08-01_at_5.12.11_PM.png]]
> >         A good way to explicitly pipe data through your UI tree, but it can become inconvenient when you need to pass some props deeply through the tree
> >     Referential Equality
> >         It defines if two values share the same reference for example `{} === {}` is false because it is checking referential equality.
> >     Memoization
> >         It is caching, In order to speed a slow running function, <u>you cache the result of it  instead of recomputing it every time</u>.
> >     The most basic form of <u>memoization</u> in react, The syntax for this hook is actually the exact same as `useEffect` since they both work in a similar way.
> >     <u>It is used to to cache a value to don’t recompute it every time as react re-compute component every time the component renders</u>.
> > ```sql
> > const result = useMemo(() => {
> > 	return slowFunction(a);
> > }, [a])
> > ```
> >     <u>useMemo will call the function and cache</u> the value when a **dependency** change.
> > 
> > `useCallback`
> >     It works nearly identical to useMemo since it will be used to cache a result based on an array of dependencies, but this hook is used for caching functions instead of caching values.
> > ```sql
> > const result = useCallback(() => {
> > 	return slowFunction(a);
> > }, [a]);
> > ```
> >     useCallback <u>will not call the function</u> instead it will **return a new version of the function **so if no dependency change, it will maintain the <u>referential integrity</u> of that function.
> > 
> > `React.memo`
> >     This behaves very similar to `React.PureComponent` is that a component wrapped in React.memo will <u>not re-render unless one of the props change</u>.
> > ```sql
> > React.memo(function Component(props) {
> > 	// Code
> > });
> > ```
> > 
> > `Reducer`
> >     <u>Reducers are a different way to handle the state</u>.
> >     **Reducers must be pure.** Similar to [state updater functions](https://react.dev/learn/queueing-a-series-of-state-updates), reducers run during rendering (<u>Actions are queued until the next render</u>.)
> >     To consolidate all state updates logic outside your component in a single function 
> >     You can use `useImmerReducer` Under the hood, **Immer** will create a <u>copy</u> of your **state** with the changes you made to the `draft`
> >     You can move all the state logic into a single function outside your component 
> >     To create a `reducer`, move all the setState logic into `dispatch`, this way is different than setState, this way you are telling what the user just did to dispatch “actions” from the event handlers (The update logic will live in another place)
> >     The object you pass to dispatch is called an **action**.
> >     then the reducer function is where you will put your state logic, it takes two arguments, the *currentState* and the *action* object and it <u>returns the next state</u>.
> >     Then just hook it up in the component like
> >      `[tasks, dispatch] = useReducer(tasksReducer, initailTasks)`
> >     The `userReducer` is similar to `useState`, the **userReducer** takes a reducer function, an initial state and returns a <u>stateful value</u>, a <u>dispatch function</u> (to dispatch actions to the reducer)
> > 
> > `useId`
> >     A hook to generate an Id for a DOM element
> > 
> 
> > [!note]+ React Router
> > `**BrowserRouter**` uses both the [**history**](https://github.com/ReactTraining/history) library as well as [**React Context**](https://ui.dev/react-context). The history library helps React Router [**keep track of**](https://github.com/ReactTraining/react-router/blob/dev/packages/react-router-dom/index.tsx#L98) the browsing history of the application using the <u>browser's built-in history stack</u>, and **React Context** helps make **history** [**available wherever**](https://github.com/ReactTraining/react-router/blob/dev/packages/react-router/index.tsx#L245) React Router needs it.
> > 
> > `**MemoryRouter**` keeps track of the history of the application in memory, rather than in the URL. Use this instead of `**BrowserRouter**` if you're developing a React Native application.
> > 
> > `**StaticRouter**`, as the name implies, is useful in environments where the app's location never actually changes, like when rendering a single route to static HTML on a server.
> > 
> > `**Route**` allows you to map your app's location to different React components
> > 
> > `**Routes**` as the metaphorical conductor of your routes. Whenever you have one or more `**Route**`s, you'll most likely want to wrap them in a `**Routes**`. The reason for this is because it's `**Routes**` job is to understand all of its `**children**` `**Route**` elements, and intelligently choose which ones are the best to render
> > 
> > `**useParams**` Hook that returns an object with a mapping between the URL parameter(s) and its value.
> > 
> > `**useSearchParams**`** **query string using .
> > 
> > ## **Nested Routes**
> > 
> > Nested Routes allow the parent `**Route**` to act as a wrapper and control the rendering of a child `**Route**`.
> > 
> > By appending a `**/***` to the end of our `**/messages**` path, we're essentially telling React Router that `**Messages**` has a nested `**Routes**` component and our **parent **`**path**` should match for `**/messages**` as well as any other location that matches the `**/messages/***` pattern. Exactly what we wanted.
> > 
> > ```sql
> > // App.js
> > function App() {
> >   return (
> >     <Routes>
> >       <Route path="/" element={<Home />} />
> >       <Route path="/messages/*" element={<Messages />} />
> >       <Route path="/settings" element={<Settings />} />
> >     </Routes>
> >   );
> > }
> > 
> > 
> > // Messages
> > function Messages() {
> >   return (
> >     <Container>
> >       <Conversations />
> > 
> >       <Routes>
> >         <Route path=":id" element={<Chat />} />
> >       </Routes>
> >     </Container>
> >   );
> > }
> > ```
> > 
> > Also, you can do it like 
> > 
> > ```sql
> > // App.js
> > <Route path="/messages" element={<Messages />}>
> >         <Route path=":id" element={<Chats />} />
> > </Route>
> > 
> > // Messages
> > import { Outlet } from "react-router-dom";
> > 
> > function Messages() {
> >   return (
> >     <Container>
> >       <Conversations />
> > 
> >       <Outlet />
> >     </Container>
> >   );
> > }
> > ```
> > 
> > React Router offers two different ways to programmatically navigate, depending on your preference. First is the [**imperative**](https://ui.dev/imperative-vs-declarative-programming) `**navigate**` method and second is the declarative `**Navigate**` component.
> > 
> > To catch “404” pages
> > 
> > ```sql
> > <Routes>
> >   <Route path="*" element={<NotFound />} />
> >   <Route path="/" element={<Home />} />
> >   <Route path="/about" element={<About />} />
> >   <Route path="/settings" element={<Settings />} />
> > </Routes>
> > ```
> > 
> > In link Anytime you pass data along via the `**state**` prop, that data will be available on the `**location**`'s `**state**` property, which you can get access to by using the custom `**useLocation**` Hook that comes with React Router.
> > 
> > 
> 
> > [!note]+ Forms
> > Controlled Forms (Not recommended)
> >     When you useState for your forms (You let React control your forms)
> > 
> > Uncontrolled Forms
> >     When you let the browser control the form inputs and you grab them by the formData
> 
> > [!note]+ Lazy Loading (Code Splitting)
> > A way to only load the route that the user is into, vite will look at these lazies and will not include them in the first page load.
> > 
> > to lazy import the component 
> > 
> > ```typescript
> > import { lazy } from "react"
> > 
> > const Details = lazy(() => import('./Details'));
> > ```
> > 
> > Then you need to wrap your app inside a Suspense component
> > 
> > ```typescript
> > import { Suspense, lazy }
> > 
> > 
> > const Details = lazy(() => import('./Details'));
> > 
> > // App return
> > // .. Code
> > 	<Suspense fallback={<div> Loading... </div>}
> > 		// .. Code
> > 	</Suspense>
> > ```
> 
> > [!note]+ React.Portal 
> > Allows you to render to a different place from the component
> 
> > [!note]+ React Query
> > Asynchronous state management for TS/JS React library
> > 
> 

> [!note]+ State management
> State management is like keeping track of everything happening in your application.
> 
> There are two types of state management
> 
> > Local state management deals with data specific to a single component, while global state management deals with data that needs to be shared and accessed by multiple components throughout the application.
> 
> > [!note]+ Local
> > It typically involves managing the data within a single component or module of an application. This local state is specific to that component and is not directly accessible or modifiable by other parts of the application unless explicitly passed down as props or through other means.
> > 
> > ```javascript
> > import React, { useState } from "react";
> > 
> > const Counter = () => {
> >   // Local state variable 'count'
> >   const [count, setCount] = useState(0);
> > 
> >   // Update local state 'count'
> >   const increment = () => {
> >     setCount(count + 1);
> >   };
> > 
> >   return (
> >     <div>
> >       <p>Count: {count}</p>
> >       <button onClick={increment}>Increment</button>
> >     </div>
> >   );
> > };
> > 
> > export default Counter;
> > ```
> 
> > [!note]+ Global
> > It involves managing data that needs to be accessed and modified by multiple components across the application. This global state is typically stored in a centralized location, such as a global store or context, where it can be accessed and modified by any component that needs it without the need for prop drilling or passing data through multiple layers of components.
> > 
> > ```javascript
> > import React, { createContext, useContext, useState } from "react";
> > 
> > // Create a global context
> > const CounterContext = createContext();
> > 
> > // Create a provider to manage global state
> > const CounterProvider = ({ children }) => {
> >   // Global state variable 'count'
> >   const [count, setCount] = useState(0);
> > 
> >   // Update global state 'count'
> >   const increment = () => {
> >     setCount(count + 1);
> >   };
> > 
> >   return (
> >     <CounterContext.Provider value={{ count, increment }}>
> >       {children}
> >     </CounterContext.Provider>
> >   );
> > };
> > 
> > // Custom hook to access global state and updater
> > const useCounter = () => {
> >   return useContext(CounterContext);
> > };
> > 
> > export { CounterProvider, useCounter };
> > ```
> > 
> > 
> 
> > [!note]+ Some Libraries
> > - [**Redux**](https://redux.js.org/): Redux helps organize your app's data in one place using actions and reducers.
> > - [**Zustand**](https://zustand-demo.pmnd.rs/): Zustand is a simple way to handle your app's data using React hooks.
> > - [**Context API**](https://react.dev/reference/react/useContext): Context API shares data across your app without needing extra libraries.
> > - [**Recoil**](https://recoiljs.org/): Recoil makes managing data in React easier with atoms and derived states.
> > - [**MobX**](https://mobx.js.org/README.html): MobX makes it easy to watch and update your data without extra setup.
> 
> <u>**Ask yourself these questions before**</u>
> 
> - **What kind of data are we displaying in this application?** 
> - **What kind of data is repeated on different pages?** 
> - **Is there any specific data being used across the entire application?**
> 

> [!note]+ **URL State Management**
> A URL (Uniform Resource Locator) with parameters typically consists of several components:
> 
> 1. **Scheme**: Specifies the protocol used to access the resource, such as **http://** or **https://**.
> 2. **Domain**: The domain name or IP address of the server hosting the resource.
> 3. **Port**: (Optional) Specifies the port number to which the request should be sent. Default ports are often omitted (e.g., port **80** for HTTP, port **443** for HTTPS).
> 4. **Path**: The specific resource or endpoint on the server, typically represented as a series of directories and filenames.
> 5. **Query Parameters**: (Also known as **searchParams** in Next.js) Additional data sent to the server as part of the request, typically used for filtering or modifying the requested resource. Query parameters are appended to the URL after a question mark **"?"** and separated by ampersands **"&"**
>     For example: **?param1=value1&param2=value2**
> 6. **Fragment**: (Optional) Specifies a specific section within the requested resource, often used in web pages to navigate to a particular section. It is indicated by a hash **"#"** followed by the fragment identifier.
> 
> Keeping URL params in the URL as state-management
> 
> ```javascript
> alig.com/editor/xxxxx?filter=course_name%3Aeq%3Aultimate-next-js-13-course-ebook&sort=created_at%3Adesc
> ```
> 
> In simple terms, imagine your website is like a big library, and each URL is like a unique bookshelf. When you organize your books (content) on different bookshelves (URLs), it's easier for people (search engines) to find the specific book (content) they're looking for.
> 
> This organization helps improve your website's visibility and ranking in search engine results because search engines can better understand the structure and relevance of your content.
> 
> Using unique URLs for different states or pages also makes it easier for users to share specific content and for search engines to understand the context of that content, which ultimately can lead to better search engine rankings for your website.
> 
> 💡 **How does URL as a state management improve SEO?**
>     When you use unique URLs to represent different states or pages within your website, each URL serves as a distinct entry point for search engines to crawl and index your content.
>     This means that all the variations of your content, such as different filters, sorting options, or pagination, have their own URLs, making it easier for search engines to discover and rank your pages appropriately.
> 
> We can manage it in a very nice way for the filters too using [query-string](https://www.npmjs.com/package/query-string) NPM package
> 
> ```javascript
> export function formUrlQuery({ params, key, value }: UrlQueryParams) {
>   const currentUrl = qs.parse(params);
> 
>   currentUrl[key] = value;
> 
>   return qs.stringifyUrl(
>     {
>       url: window.location.pathname,
>       query: currentUrl,
>     },
>     { skipNull: true },
>   );
> }
> 
> const handleUpdateParams = (value: string) => {
>   const newUrl = formUrlQuery({
>     params: searchParams.toString(),
>     key: "location",
>     value,
>   });
> 
>   router.push(newUrl);
> };
> 
> ```


<!-- Column 2 -->


> [!note]+ Redux
> Redux under the hood uses the <u>Context API</u> to give all your components access to the **store**.
> 
> *You can use Redux not only with React*, you can use it anywhere where **JS** is found
> 
> <u>Redux toolkit is an abstraction on top of React-Redux</u>.
> 
> `React-Redux` is the connector between redux and react.
> 
> Only put the <u>business state</u> inside Redux, **don’t** put the **UI** state there.
> 
> Redux has <u>**5**</u> (9 Really) Functions
>     `compose`
>         <u>Function allows you to combine functions</u> and pass data for both of them and create a new function that **combines** them. It creates an array of the functions and use `reduce` on them and <u>call the functions passing the param to each one</u>.
> ```typescript
> const makeLouder = (string) => string.toUpperCase();
> const repeatThreeTimes = (string) => string.repeat(3);
> const embolden = (string) => string.bold();
> 
> const combinedFunction = compose(repeatThreeTimes, makeLouder, embolden)
> 
>  /* compose(repeatThreeTimes, makeLouder, embolden)
>  * This is under the hood
>  * repeatThreeTimes(
>  *    makeLouder(
>  *      embolden()
>  *    )
>  * )
>  */
> ```
>     `combineReducers`
>         <u>reducer is just a function that takes a state and an action and output a new state</u>.
> ```typescript
> type ApplicationState = {};
> type Action = { type: string; [key: string]: any };
> 
> type Reducer = (state: ApplicationState, action: Action) 
> 						 => ApplicationState;
> ```
> ![[Screenshot_2023-08-02_at_1.47.20_PM.png]]
> 
>     `createStore` (This has 4 more functions xD)
>         A store is the place where we keep our state, it is an <u>anti-pattern that you have multiple stores</u>.
>         It takes 3 arguments (last 2 is optional) (Reducer, Initial State, Store Enhancer) 
>         <u>A store is a global container for state of your application</u>.
> ```typescript
> import {
>   createStore,
>   compose,
>   applyMiddleware,
>   bindActionCreators
> } from "redux";
> 
> const initState = { value: 0 };
> 
> const CounterActions = {
>   INCREMENT: "INCREMENT"
> }
> 
> const increment = (amount) => {
>   return {
>     type: CounterActions.INCREMENT,
>     payload: amount,
>     meta: {},
>     error: ""
>   };  
> }
> 
> const reducer = (state = initState, action) => {
>   switch (action.type) {
>     case CounterActions.INCREMENT:
>       return { state: state.value + action.payload };
>     default:
>       return state;
>   }
> };
> 
> const store = createStore(reducer);
> ```
>         It has 4 methods in the store
>             `dispatch`
>                 <u>is used to emit actions to the current reducer</u>, only one requirement for an action and that it has a **type**.
>                 We use <u>Flux Types</u> (type, payload, meta, error)
> ```typescript
> const incrementAction = {
>   type: 'INCREMENT',
>   payload: 5,
>   meta: { /** aditional information about the action */ },
>   error: ''
> };
> 
> store.dispatch(incrementAction);
> ```
>             `subscribe`
>                 `subscribe` method. This method takes a function dictates what should happen whenever the state in the store is updated.
>                 `subscribe` also returns a function that you can call to unsubscribe.
> ```typescript
> const subscription = () => console.log('Subscribed', store.getState().value);
> const unsubscribe = store.subscribe(subscription);
> ```
>             `getState`
>                 This function returns the current state
>             `replaceReducer`
>                 This replace the current Reducer for a new reducer. This is useful for code splitting.
>     `bindActionCreators`
>         It gives us a way to combine our action creators together and call dispatch automatically on them.
>         <u>under the hood it uses compose</u>
>             compose(store.dispatch, increment);
> ```typescript
> import {
>   createStore,
>   compose,
>   applyMiddleware,
>   bindActionCreators
> } from "redux";
> 
> const initState = { value: 0 };
> 
> const CounterActions = {
>   INCREMENT: "INCREMENT",
>   MULTIPLY: "MULTIPLY"
> }
> 
> const increment = (amount) => {
>   return {
>     type: CounterActions.INCREMENT,
>     payload: amount,
>     meta: {},
>     error: ""
>   };  
> }
> 
> const multiply = (amount) => {
>   return {
>     type: CounterActions.MULTIPLY,
>     payload: amount,
>     meta: {},
>     error: ""
>   };  
> }
> 
> const reducer = (state = initState, action) => {
>   switch (action.type) {
>     case CounterActions.INCREMENT:
>       return { value: state.value + action.payload };
>     default:
>       return state;
>   }
> };
> 
> const store = createStore(reducer);
> 
> const actions = bindActionCreators({
>   increment,
>   multiply
> }, store.dispatch);
> 
> actions.multiply(4);
> actions.increment(400);
> ```
>     `combineReducer`
>         Will combine the reducers into one reducer, the advice here the flatter your object is going to be the better,
>         every one of the combined reducers will get the action passed here
> ```typescript
> const ADD_TASK = "ADD_TASK";
> const ADD_USER = "ADD_USER";
> 
> const initState = {
>   users: [
>     { id: 1, name: "Steve" },
>     { id: 2, name: "Ali" }
>   ],
>   tasks: [{ title: "File Something up" }, { title: "Order more cupcakes" }]
> };
> 
> const addTask = (title) => ({ type: ADD_TASK, payload: title });
> const addUser = (name) => ({ type: ADD_USER, payload: name });
> 
> const userReducer = (state = initState.users, action) => {
>   if (action.type == ADD_USER) {
>     return [...state, action.payload];
>   }
> 	return state;
> };
> 
> const taskReducer = (state = initState.tasks, action) => {
>   if (action.type == ADD_TASK) {
>     return [...state, action.payload];
>   }
> 	return state;
> };
> 
> const reducer = combineReducers({ users: userReducer, tasks: taskReducer });
> const store = createStore(reducer);
> ```
>     `storeEnhancer`
> ![[Screenshot_2023-08-02_at_2.53.24_PM.png]]
>         A function that allows us to <u>modify</u> and <u>extend</u> on the `createStore`, it will gives us a store <u>**object**</u> to play with.
> ```typescript
> const basicReducer = (state) => state;
> 
> const monitorEnhancer = (createStore) => (reducer, initState, enhancer) => {
>   const monitorReducer = (state, action) => {
>     const start = performance.now();
>     const newState = reducer(state, action);
>     const end = performance.now();
>     const diff = Math.round(end - start);
>     console.log("Performance: " + diff + " ms");
>     return newState;
>   };
> 
>   return createStore(monitorReducer, initState, enhancer);
> };
> 
> const store = createStore(basicReducer, {}, monitorEnhancer);
> store.dispatch({ type: "Hello" });
> ```
>         Another one with combineReducers
> ```typescript
> import {
>   createStore,
>   compose,
>   applyMiddleware,
>   bindActionCreators,
>   combineReducers
> } from "redux";
> 
> const initState = {
>   users: [{ id: 1, name: "Ali" }],
>   tasks: [{ title: "First Task" }]
> };
> 
> const ACTION_ADD_TASK = "ACTION_ADD_TASK";
> const ACTION_ADD_USER = "ACTION_ADD_USER";
> 
> const tasksReducer = (state = initState.tasks, action) => {
>   if (action.type === ACTION_ADD_TASK) {
>     return [...state, action.payload];
>   }
>   return state;
> };
> 
> const usersReducer = (state = initState.users, action) => {
>   if (action.type === ACTION_ADD_USER) {
>     return [...state, action.payload];
>   }
>   return state;
> };
> 
> const combinedReducer = combineReducers({
>   tasks: tasksReducer,
>   users: usersReducer
> });
> const stateEnhancer = (createStore) => (reducer, initState, enhancer) => {
>   const stateLogger = (state, action) => {
>     console.log("Previous State: ", state, "Action: ", action);
>     const newState = reducer(state, action);
>     console.log("New State: ", newState, "Action: ", action);
>     return newState;
>   };
> 
>   return createStore(stateLogger, initState, enhancer);
> };
> 
> const monitorEnhancer = (createStore) => (reducer, initState, enhancer) => {
>   const monitorReducer = (state, action) => {
>     const start = performance.now();
>     const newState = reducer(state, action);
>     const end = performance.now();
>     const diff = Math.round(end - start);
>     console.log("Performance: " + diff + " ms");
>     return newState;
>   };
> 
>   return createStore(monitorReducer, initState, enhancer);
> };
> 
> const store = createStore(combinedReducer, initState, compose(monitorEnhancer, stateEnhancer));
> store.dispatch({
>   type: ACTION_ADD_TASK,
>   payload: {
>     title: "New Task"
>   }
> });
> ```
>     `applyMiddleware`
>         A way to manage actions as they pass through and to compose an entire chain of enhancers. for example to log data to analytics report
> ```typescript
> const initState = {
>   users: [{ id: 1, name: "Ali" }],
>   tasks: [{ title: "First Task" }]
> };
> 
> const ACTION_ADD_TASK = "ACTION_ADD_TASK";
> const ACTION_ADD_USER = "ACTION_ADD_USER";
> 
> const tasksReducer = (state = initState.tasks, action) => {
>   if (action.type === ACTION_ADD_TASK) {
>     return [...state, action.payload];
>   }
>   return state;
> };
> 
> const usersReducer = (state = initState.users, action) => {
>   if (action.type === ACTION_ADD_USER) {
>     return [...state, action.payload];
>   }
>   return state;
> };
> 
> const combinedReducer = combineReducers({
>   tasks: tasksReducer,
>   users: usersReducer
> });
> 
> const logMiddleWare = store => next => action => {
>   console.log('Old state', store.getState(), action);
>   next(action);
>   console.log('new state', store.getState(), action);  
> }
> 
> const monitorMiddleWare = state => next => action => {
>   const performanceNow = performance.now();
>   next(action);
>   const performanceAfter = performance.now();
>   const difference = Math.round(performanceAfter - performanceNow);
>   console.log('Performance after state change: ' + difference + ' ms')
> };
> 
> const store = createStore(combinedReducer, initState,
> 	 applyMiddleware(logMiddleWare, monitorMiddleWare));
> store.dispatch({
>   type: ACTION_ADD_TASK,
>   payload: {
>     title: "New Task"
>   }
> });
> ```
> 
> `useSelector`
>     This hook allows you to grab a <u>state object from the state itself</u> (select from the state)
> ```typescript
> const count = useSelector(state => state.count);
> ```
> 

> [!note]+ Redux Toolkit
> <u>Unlike hooks, it is not tight into the React component life cycle</u>
> 
> <u>It is an abstraction on top of React-Redux</u>
> 
> It uses <u>**Immer**</u> *under the hood* with **reducers** so it can <u>manipulate</u> a `draft` (proxy) and send a new state out.
> 
> `configureStore`
>     Creates a store, Here it will apply 3 middlewares
>         1. Checks if you are mutating the state somewhere
>         2. Redux Thunk
>         3. Redux Dev Tools
> ```typescript
> export const store = configureStore({
> 	name: 'Tasks',
> 	initialState: initState,
>   reducer: {
>     tasks: taskSlice.reducer,
>     humans: humanSlice.reducer
>   }
> });
> ```
> 
> `createSlice`
>     <u>Think of it as a piece of feature</u>
>     It combines **all** your <u>reducers</u>, <u>action creators</u> and <u>actions</u> inside it
>     It takes a name to differentiate between <u>slices</u>
>     It will create an action<u> on the behalf of us</u>
> ```typescript
> import { createSlice, nanoid } from '@reduxjs/toolkit';
> 
> const createTask = (title) => ({
>   id: nanoid(),
>   title,
>   completed: false,
>   assignedTo: ''
> });
> 
> const initState = [createTask('Order Coffee'), createTask('Study Redux')];
> 
> export const taskSlice = createSlice({
>   name: 'Tasks',
>   initialState: initState,
>   add: (state, action) => {
>     const task = createTask(action.payload);
>     state.push(task);
>   }
> });
> ```
>     But what if we want an <u>action</u> that is not **automatically generated** ?
>     we use `createAction`
>         <u>It formates the action that you pass to dispatch</u>
> ```typescript
> export const toggle = createAction('tasks/toggle',
> (taskId, completed) => ({
>   payload: taskId,
>   completed
> }));
> ```
>     Now, what if we need to notify another slice about a change in our slice ?
>     You use the `extraReducers`
>         <u>It creates a reducers that can listen to an events in other slices</u>
> ```typescript
> export const humanSlice = createSlice({
>   name: 'Humans',
>   initialState: initState,
>   reducers: {
>     add: (state, action) => {
>       state.push(createHuman(action.payload));
>     }
>   },
>   extraReducers: (builder) => {
>     builder.addCase(taskSlice.actions.assignToUser, (state, action) => {
>       for (const human of state) {
>         if (human.id === action.payload.humanId) {
>           human.taskIds.push(action.payload.taskId);
>         } else {
>           human.taskIds.filter((taskId) => taskId !== action.payload.taskId);
>         }
>       }
>     });
>   }
> });
> ```
>     `createReducer`
>         This will take the <u>initial state</u> and a <u>builder</u>, like creating the extra reducers, it has a 
>             `addCase`
>                 Like the switch statement case, it will switch upon the action type
>             `addDefaultCase`
>                 This if you want to catch the default (Return the state maybe) it will automatically do that for you with the beautiful builder
>             `addMatcher`
>                 Allows you to match your incoming actions against your own filter function instead of only the `action.type` property
> 
>     `reduxThunk`
> ![[Screenshot_2023-08-02_at_9.04.06_PM.png]]
>         <u>It is a simple way to hit an API</u>, but if you want to use <u>debouncing</u> and <u>throttling</u> **use** **Redux Saga or Redux Observables**.
>         It allows you to <u>dispatch</u> functions that will <u>eventually dispatch actions</u>.
>         `createAsyncThunk`
>             Is for creating that thunk will eventually have the data we need
>             It will start by firing up a `pending` event stating that it has started the request,
>             then it will fire an event with `fulfilled` or `rejected`
> ```typescript
> import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';
> 
> const initialState = {
>   data: [],
>   loading: false,
> };
> 
> const ENDPOINT = 'https://star-wars-characters.glitch.me/api/search/';
> 
> export const fetchCharacherFromAPI = createAsyncThunk(
>   'characters/fetchCharacters',
>   async (searchTerm) => {
>     const response = await fetch(ENDPOINT + searchTerm).then((response) =>
>       response.json(),
>     );
>     return response.results;
>   },
> );
> 
> export const charactersSlice = createSlice({
>   name: 'characters',
>   initialState,
>   reducers: {
>     add: (state, action) => {
>       state.characters = action.payload;
>     },
>   },
>   extraReducers: {
>     [fetchCharacherFromAPI.fulfilled]: (state, action) => {
>       state.data = action.payload;
>       state.loading = false;
>     },
>     [fetchCharacherFromAPI.pending]: (state, action) => {
>       state.loading = true;
>     },
>   },
> });
> ```
> 
> `TypedUseSelectorHook`
>     This interface allows you to easily create a hook that is properly typed for your
>  store's root state.
> 
> A great type 
> 
> ```typescript
> type RequireOnly<T, P extends keyof T> = Pick<T, P> & Partial<Omit<T, P>>;
> 
> // Then you can use it 
> type DraftUser = RequireOnly<User, 'realName' | 'alterEgo'>
> ```
> 
> In your store.ts
> 
> ```typescript
> import { configureStore } from "@reduxjs/toolkit";
> import { tasksReducer } from "../features/tasks-slice";
> 
> const store = configureStore({
>     reducer: {
>         tasks: tasksReducer
>     }
> });
> 
> export type ApplicationState = ReturnType<typeof store.getState>;
> export type ApplicationDispatch = typeof store.dispatch;
> 
> export default store;
> ```
> 
> And in your hooks.ts
> 
> ```typescript
> import { useSelector, TypedUseSelectorHook, useDispatch,  } from "react-redux";
> import { ApplicationDispatch, ApplicationState } from './store';
> 
> 
> export const useAppSelector: TypedUseSelectorHook<ApplicationState> = useSelector;
> export const useAppDispatch: () => ApplicationDispatch = useDispatch
> ```
> 
> Then you can just use them like
> 
> ```typescript
> const tasks = useAppSelector(state => state.tasks.entities);
> const dispatch = useAppDispatch();
> ```
> 
> RTK Query
>     <u>You can use without Redux Toolkit</u>, This is built on top of **redux**, it gives us so much power over <u>Redux Thunk</u>.
>     `createApi`
>         It takes what the reducer path that you want
>         it will give it what the baseQuery will use (fetch or axios) with the baseUrl
>         and the endpoints
> ```typescript
> export const itemApi = createApi({
>     reducerPath: 'itemsApi',
>     baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
>     endpoints: (builder) => {
>         return {
>             getItems: builder.query<Item[], void>({
>                 query: () => 'items'
>             })
>         }
>     }
> })
> ```
>         if you got an existing store
> ```typescript
> import { configureStore, getDefaultMiddleware } from "@reduxjs/toolkit";
> import { itemApi } from "./services/api-service";
> 
> const store = configureStore({
>     reducer: {
>         [itemApi.reducerPath]: itemApi.reducer
>     },
>     middleware: (getDefaultMiddleware) => {
>         return getDefaultMiddleware().concat(itemApi.middleware)
>     }
> });
> 
> export default store;
> ```
> 

> [!note]+ React Performance
> ![[Screenshot_2023-08-03_at_7.06.34_PM.png]]
> 
> Caching and Memoizing Components
> 
> Component Structure
> 
> Concurrency with useTransition useDeferredValue
> 
> ![[Screenshot_2023-08-03_at_8.16.52_PM.png]]
> 
> A golden rule: Not doing stuff is faster than doing stuff, and doing stuff now is better than doing them later.
> 
> React Rendering Cycle
> ![[Screenshot_2023-08-03_at_8.18.59_PM.png]]
>     Render Phase ⇒ Commit Phase ⇒ Clean up Phase 
>     Render Phase
> ![[Screenshot_2023-08-03_at_8.24.21_PM.png]]
> ![[Screenshot_2023-08-03_at_8.39.18_PM.png]]
>     If the component doesn’t exist, React don’t have to do diffing, just put it on the page, but if it exists it will do diffing.
>     When something change it goes through the Render phase, it calls all the function, classes and see what is difference from the previous time (last time it render) then it figure out what is the best changes to the DOM and do them.
>     `useState` is an abstraction over `useReducer`
>     It started with a state change, then it goes to the render cycle.
>     if you trigger multiple state changes (If you are using 17 and earlier) some times they will be patched (Only in that function) if it has a setTimeout or a promise no longer will it be patched.
>     In React 18 you get automatic patching, only when it finishes it will go with the next round of state changes.
> ![[Screenshot_2023-08-03_at_8.52.59_PM.png]]
>     If you need to get out of the atomic batching use `flushSync`
> 
>     Under the hood from React v16 and later, React Fiber has become a thing
> ![[Screenshot_2023-08-03_at_8.42.54_PM.png]]
> 
>     There’s strategies with this tree.
>         From the point of the state change and down will get rendered.
> 
>     Commit Phase
>     Then it takes a list of changes that needed to update the DOM and update them in the commit phase and sweeps to see if all refs are still pointing to the same objects.
>     In between the commit and cleanup phase passel effect phase where the useEffect happens
>     Cleanup phase
> 
> 
> 
> 
> 

> [!note]+ React Reconciliation & Fiber
> [https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
> 
> Fiber uses a LinkedList
> 
> The mechanism of tracking changes and applying them to the screen is known as reconciliation. 
> 
> VDOM is the tree of immutable elements returned from the render method of react.
> 
> besides the VDOM tree (React element tree) it also has a tree that keeps track of the components, DOM nodes and used to keep the state.
> 
> During reconciliation data from every react element returned from the render() method is merged into the tree of fiber nodes. 
> 
> Every React element has a cor-responding fiber node, unlike react-elements fibers aren’t re-created on every render, These are mutable data structures that hold components state and DOM.
> 
> **You can think of a fiber as a data structure that represents some work to do or, in other words, a unit of work. Fiber’s architecture also provides a convenient way to track, schedule, pause and abort the work.**
> 
> When a react element is converted into a fiber node for the first time, React uses the data from the element to create a fiber in the [createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414) function.
> 
> When react ends up with a fiber tree that reflects the state of the application that was render used to render the UI, this tree is called current.
> 
> Then when it starts working on the update it builds a so-called workInProgress tree that reflects the future state to be flushed to the screen.
> 
> All work is performed on fibers is from the workInProgress tree.

> [!note]+ Critical Rendering Path
> By definition, **Critical Rendering Path **is nothing but a collection of steps that make a path between the time your browser gets an HTML page and starts building the webpage for users to visualize. During this process, you need to optimize the steps your browser does.
> 
> Critical Resource: All resources which blocks the rendering of the page.
> 
> Critical Path Length: Total number of round trips required to fetch all the critical resources required for building the page.
> 
> Critical Bytes: Total number of bytes transferred as a part of completing and building the page.
> 
> **Async, Defer, Preload**
>     **Note**: Async, and Defer are attributes to be used on external scripts.
> > [!note]+ Async 
> > ![[Untitled 48.png]]
> > 
> > It allows the browser to do something else, while the JavaScript resource gets downloaded the already downloaded JS resource will be executed, as soon as the download complete.
> > 
> > 1. JS is async downloaded.
> > 2. Execution of all other scripts will be paused
> > 3. DOM rendering will happen simultaneosuly
> > 4. DOM rendering will pause only when script is executed
> > 5. Rendering  blocking JS issues can be solved with async
> 
> > [!note]+ Defer
> > ![[Untitled 49.png]]
> > 
> > When you use it, JS will be downloaded while the HTML rendering happens, and the execution will be done when the HTML file is completely rendered.
> 
> > [!note]+ Preloaded
> > When you use preload, the browser will download the resource and the execution will happen the moment the resource is available.
> 
> Caching 
>     To achieve caching in the Front-end code, the browser look for four important header attributes in the HTTP response
>         1. ETag (Entity Tag)
> A string which validates the cache token, this token can be used by the browsers to decide if the request can be satistfied with a copy in the cache or not, if the resource didn’t change the server will return the same hash Token, and the body will be empty, if not, the body will be populated with the most recent data.
>         2. Cache-Control
> allows your application to decide on the browser cache policy for a given request, you can choose from no-store, no-cache, private or public.
>         3. Last-Modified
> Is quite similar to ETag, but it depends on the last modified header the data and time helps the client to decide if a new request must be made or not.
>         4. Expires
> For deciding on the validity of data.
> 

> [!note]+ Webpack, React Eject, Vite
> Vite uses when bundling your app: 
>     **Tree Shaking** (Live Code Inclusion) 
>         Only includes code that you are actively using
>     **Dead Code elimination**
>         Find code that was never called and deletes them

> [!note]+ SSR
> Is a rendering technique where we render client-side single page application on the server and then send a fully rendered page to the client.
> 
> In server-side rendering, when a user requests a webpage, the server prepares an HTML page by fetching user-specific data and sends it to the user’s machine. The browser then constructs the content and displays the page. This entire process of fetching data from the database, creating an HTML page and sending it to the client happens in a few milliseconds.
> 
> ![[Untitled 50.png]]
> 
> Hydration
>     Is a technique used similar to rendering but instead of having and empty DOM to render all of our component into we have a DOM that was pre-built for us, we tell here react to attach event handlers to the HTML to make the app interactive, this process of rendering our components and attaching event handlers is known as hydration.
> 
> Why SSR:
>     1. Improves Time to First Byte where the browsers can start displaying the fetched HTML from the server without waiting for all the JS to be downloaded and executed.
>     2. It improves SEO because the content is directly available not requiring JS execution to view it.
>     3. Better Initial page loading (You see a fully populated page)
>     4. Access to server-side resource, you can access any server resources directly when generating the content.
> 
> How SSR Works:
>     1. The server reads the request and determines the appropriate content to load
>     2. It executes the React server-side component to generate the HTML content.
>     3. The server sends the HTML to the client
>     4. The client displays the HTML content while it is downloading the JS bundle.
>     5. Once the JS is downloaded it takes over and the SPA “Hydrates” to become fully interactive.

> [!note]+ Progressive Web Apps (PWAs)
> Type of web application designed to work offline, perform well and provide a native-app-like experience.
> 
> Progressive → Works for every user, regardless the browser
> 
> Responsive, Connectivity independent.
> 
> Server workers are the backbone of PWA’s they are JS files that can control the web app it is associated with, intercepting and modifying navigation and resource request and caching resources, Service workers works separately from the main browser thread, enabling them to work offline.
> 
> Service worker life-cycle
>     1. Registration
> The service worker is registered using JS on the web app, once registered the browser tries to install it
>     2. Installation
> The install event, you can cache some static assets, once everything is cached successfully the service worker becomes installed.
>     3. Activation
> Once installed, the service worker activates, this is where you manage the old caches.
>     4. Idle
> When it is not processing events
>     5. Termination
> To save memory, an idle service worker is terminated
>     6. Fetch/Message
> Whenever there’s a fetch or message event, the service worker wakes up to handle it.

> [!note]+ Cookies Vs Local Storage Vs Session Storage
> Cookies (Authentication, Small amount of data that should persist beyond a single session)
>     Small pieces of data that are stored on the user’s web browser.
>     Cookies has expiration property setting after a specific time, good for short-lived session and persisted data storage
>     Secure only flag ensures the http only is sent over HTTP and makes the cookie not accessed by JS preventing XSS.
>     Domain and path: Cookies are limited to 4KB each.
>     Cookies can be restricted to a particular domain or path.
>     All the cookies are sent with HTTP request from that domain.
> 
> Session Storage (Client state for one session)
>     Just like local storage but with limited lifecycle, the data is cleared when a page session ends (Page is closed)
> 
> Local Storage (Client state)
>     Web Storage solution for storing key-value pairs in a web browser with no expiration time.
>     The data inside isn’t secure, and doesn’t expire after a session ends, it has a 5 MB of limit. and it is restricted to the same origin.
> 

> [!note]+ Pure, Static, Controlled Uncontrolled, Stateful, Stateless 
> - **Pure Components**: Only re-render when state or props change.
> - **Static Components**: Always render the same thing.
> - **Controlled Components**: React controls the values of form elements.
> - **Uncontrolled Components**: DOM controls the values of form elements.
> - **Stateful Components**: Manage their own state.
> - **Stateless Components**: Do not have their own state.
> 
> Pure Components
>     Component that renders the same output for the same input and state, React.PureComponent performs a shallow comparison on the component props and state, if nothing has changed it prevents the re rendering of the component
> 
> Static Component
>     A static component does not have any state, does not have any lifecycle methods and always renders the same content based on the same props
> 
> Controlled Component
>     In React is a component where react is in control of the values of input elements and form fields, rather than leaving the DOM in control.
> 
> UnControlled Component
>     Is the one where the data is handled by the DOM itself not by react, like using a ref to get the current value of the input field rather than storing it in a state.
> 
> Stateful Components (Container Component)
>     Maintains its own state and logic to alter that state
> 
> Stateless Components (Presentational Components)
>     Doesn’t manage or alter its own state.

> [!note]+ Error Boundaries
> Error boundaries are React component that catches errors in their component tree, log those errors and display a fallback UI instead of the component that crashed.
> 
> Error boundaries catches errors during rendering, lifecycle methods and constructors of the whole tree below them.
> 
> Error boundaries do not catch errors inside the event handlers, async code (setTimeout), server-side rendering or errors thrown in the boundary itself.
> 
> Creation of an error boundary
>     You should create a class component that includes getDerivedStateFromError and componentDidCatch

> [!note]+ Styled Components
> Is a library for styling React components tht uses tagged template literals to style components.
> 
> It allows generation for unique class name for styles, automatic critical css picking to optimize performance
> 
> dynamic styling

> [!note]+ Testing
> > [!note]+ Jest
> > Is a testing library 
> > 
> > Unit Testing
> >     To test a single unit of code in isolation
> > 
> > Integration Testing
> >     Test how different pieces of the app works together, from function calls to user interactions
> > 
> > Component (Snapshot)
> >     Typically rendering the component and asserting on the output
> 
> > [!note]+ Cypress
> > A library of End-To-End testing in real browsers
> > 
> > It has Real-time reloading, time travel, automatic waiting, and network traffic control (You can stub network requests or delay them)
> 
> > [!note]+ A/B testing (Split testing)
> > is an experiment technique where two or more version is shown to users at random the performance of each version is measured against a specific goal, often a metric goal like clicks, purchases …etc 
> > 

> [!note]+ Articles
> [https://www.geeksforgeeks.org/what-is-the-difference-between-dom-and-bom/](https://www.geeksforgeeks.org/what-is-the-difference-between-dom-and-bom/)
> 
> [https://www.geeksforgeeks.org/difference-between-virtual-dom-and-real-dom/](https://www.geeksforgeeks.org/difference-between-virtual-dom-and-real-dom/)
> 
> [https://www.geeksforgeeks.org/reactjs-virtual-dom/](https://www.geeksforgeeks.org/reactjs-virtual-dom/)
> 
> [https://indepth.dev/posts/1498/101-javascript-critical-rendering-path](https://indepth.dev/posts/1498/101-javascript-critical-rendering-path)
> 
> [https://indepth.dev/posts/1501/exploring-how-virtual-dom-is-implemented-in-react](https://indepth.dev/posts/1501/exploring-how-virtual-dom-is-implemented-in-react)
> 
> [https://blog.logrocket.com/virtual-dom-react/](https://blog.logrocket.com/virtual-dom-react/)
> 
> [https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)
> 
> [https://indepth.dev/posts/1064/what-every-front-end-developer-should-know-about-change-detection-in-angular-and-react](https://indepth.dev/posts/1064/what-every-front-end-developer-should-know-about-change-detection-in-angular-and-react)
> 
> [https://javascript.info/event-loop#event-loop](https://javascript.info/event-loop#event-loop)
> 
> [https://www.linkedin.com/pulse/unraveling-javascript-event-loop-comprehensive-guide-michael-baker/](https://www.linkedin.com/pulse/unraveling-javascript-event-loop-comprehensive-guide-michael-baker/)
> 
> [https://indepth.dev/posts/1497/react-profiler](https://indepth.dev/posts/1497/react-profiler)
> 

> [!note]+ ## Courses 
> ### React
> 
> Maybe  (DONE)
> 
> [https://frontendmasters.com/courses/complete-react-v8/](https://frontendmasters.com/courses/complete-react-v8/)
> 
> Must (DONE)
> 
> [https://frontendmasters.com/courses/redux-fundamentals/](https://frontendmasters.com/courses/redux-fundamentals/)
> 
> Must (DONE)
> 
> [https://frontendmasters.com/courses/intermediate-react-v5/](https://frontendmasters.com/courses/intermediate-react-v5/)
> 
> Must (Done)
> 
> [https://frontendmasters.com/courses/advanced-redux/](https://frontendmasters.com/courses/advanced-redux/)
> 
> Must
> 
> [https://frontendmasters.com/courses/react-performance/](https://frontendmasters.com/courses/react-performance/)
> 
> Must
> 
> [https://frontendmasters.com/courses/hard-parts-ui-dev/](https://frontendmasters.com/courses/hard-parts-ui-dev/)
> 
> A big must
> 
> [https://www.youtube.com/watch?v=OcB3rTln-fI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3](https://www.youtube.com/watch?v=OcB3rTln-fI&list=PLvx8w9g4qv_p-OS-XdbB3Ux_6DMXhAJC3)
> 
> [https://frontendmasters.com/courses/tour-js-patterns/](https://frontendmasters.com/courses/tour-js-patterns/)
> 
> ### Node [Done]
> 
> Maybe
> 
> [https://frontendmasters.com/courses/node-js-v2/](https://frontendmasters.com/courses/node-js-v2/)
> 
> Must (DONE)
> 
> [https://frontendmasters.com/courses/api-design-nodejs-v4/](https://frontendmasters.com/courses/api-design-nodejs-v4/)
> 
> Must
> 
> [https://www.udemy.com/course/advanced-node-for-developers/](https://www.udemy.com/course/advanced-node-for-developers/)
> 
> 
> [https://ui.dev/react-context](https://ui.dev/react-context)
> 
> [https://ui.dev/why-react-hooks](https://ui.dev/why-react-hooks)
> 
> [https://ui.dev/ultimate-guide-to-execution-contexts-hoisting-scopes-and-closures-in-javascript](https://ui.dev/ultimate-guide-to-execution-contexts-hoisting-scopes-and-closures-in-javascript)
> 
> [https://medium.com/@AbidKazmi/all-react-hooks-in-one-short-4b0ed4b5a6e4](https://medium.com/@AbidKazmi/all-react-hooks-in-one-short-4b0ed4b5a6e4)
> 
> [https://medium.com/dooboolab/start-using-prisma-in-your-existing-database-with-prisma-introspection-334a96701bab](https://medium.com/dooboolab/start-using-prisma-in-your-existing-database-with-prisma-introspection-334a96701bab)
> 
> [https://jser.dev/2023-07-11-overall-of-react-internals/](https://jser.dev/2023-07-11-overall-of-react-internals/)
> 
> [https://medium.com/programming-essentials/how-to-debounce-throttle-a-callback-with-react-hooks-deb26b2fd7c](https://medium.com/programming-essentials/how-to-debounce-throttle-a-callback-with-react-hooks-deb26b2fd7c)
> 
> [https://wavez.github.io/react-hooks-lifecycle/](https://wavez.github.io/react-hooks-lifecycle/)
> 
> 
> [https://retool.com/blog/the-react-lifecycle-methods-and-hooks-explained/](https://retool.com/blog/the-react-lifecycle-methods-and-hooks-explained/)
> 
> [https://thoughtbot.com/blog/react-rendering-misconception](https://thoughtbot.com/blog/react-rendering-misconception)
> 
> [https://www.geeksforgeeks.org/difference-between-virtual-dom-and-real-dom/](https://www.geeksforgeeks.org/difference-between-virtual-dom-and-real-dom/)
> 
> [https://www.geeksforgeeks.org/what-is-the-difference-between-dom-and-bom/](https://www.geeksforgeeks.org/what-is-the-difference-between-dom-and-bom/)
> 
> [https://www.geeksforgeeks.org/explain-dom-diffing/](https://www.geeksforgeeks.org/explain-dom-diffing/)
> 
> [https://www.geeksforgeeks.org/what-is-diffing-algorithm/](https://www.geeksforgeeks.org/what-is-diffing-algorithm/)
> 
> [https://www.geeksforgeeks.org/explain-the-initial-cycle-in-react-which-happens-during-the-first-render/](https://www.geeksforgeeks.org/explain-the-initial-cycle-in-react-which-happens-during-the-first-render/)

> [!note]+ Concepts
> ```java
> console.log('Script Start'); // 1: This logs first, executed on call stack
> 
> setTimeout(() => {
>     console.log('setTimeout callback'); // 4: Gets queued in Macro Queue
> }, 0);
> 
> Promise.resolve().then(() => {
>     console.log('Promise resolved'); // 3: Gets queued in Micro Queue
> });
> 
> console.log('Script End'); // 2: This logs second, executed on call stack
> ```
> 
> How does this script get fired ?
> 
> ```java
> |-------------------|           |------------------|       |------------|        |------------|
> |   Call Stack      |           |  Web API Queue   |       |  Micro     |        |    Macro   |
> |-------------------|           |------------------|       |  (Job)     |        |   (Task)   |
> |                   |           |                  |       |  Queue     |        |   Queue    |
> |   console.log     |           |  setTimeout      |       |------------|        |------------|
> |   (Script Start)  |           |  (0 seconds)     |       | Promise    |        | setTimeout |
> |-------------------|           |------------------|       | .then()    |        | callback   |
> |                   |           |                  |       |------------|        |------------|
> |   console.log     |           |                  |       |            |        |            |
> |   (Script End)    |           |                  |       |            |        |            |
> |-------------------|           |------------------|       |------------|        |            |
> 
> |-------------------|
> |   Event Loop      |
> |-------------------|
> |                   |
> |  Constantly       |
> |  Monitoring       |
> |-------------------|
> ```
> 
> 1. The Call Stack executes `**console.log('Script Start')**` directly.
> 2. The `**setTimeout**` function is encountered and its callback is handed off to the Web API Queue. Web API takes care of the timer.
> 3. While the timer is counting (even though it's 0 seconds), the Call Stack moves on and executes the `**Promise.resolve()**`, placing its `**.then()**` callback in the Micro Queue.
> 4. The Call Stack then executes `**console.log('Script End')**`.
> 5. The Event Loop, seeing the Call Stack is empty, checks the Micro Queue first and executes the Promise's `**.then()**` callback (`**console.log('Promise resolved')**`).
> 6. After the Micro Queue is emptied, the Event Loop checks the Macro Queue and executes the `**setTimeout**` callback (`**console.log('setTimeout callback')**`).
> 
> 7. **Call Stack**:
>     - It's a mechanism for JavaScript to keep track of its function execution context.
>     - When you invoke a function, its details (like parameters, local variables) get pushed onto the stack. Once the function completes its execution, it gets popped off the stack.
>     - It's a LIFO (Last In First Out) data structure, meaning the last function that gets pushed onto the stack is the first to be popped out when it completes.
> 8. **Event Loop**:
>     - The Event Loop's job is to constantly check the Call Stack and see if there's any function that needs to run.
>     - If the Call Stack is empty, it will check the message queues to see if there are any pending callbacks waiting to be executed.
>     - Essentially, it ensures that JavaScript remains non-blocking, as it constantly checks if there's asynchronous code that's ready to be executed.
> 9. **Macro Queue (Task Queue)**:
>     - It's a queue where asynchronous callbacks (like those from `**setTimeout**` or `**setInterval**`) are placed after their time requirement has been met but are waiting to be executed.
>     - The event loop checks this queue once the call stack is empty.
> 10. **Micro Queue (Job Queue)**:
>     - This is another queue, but with a higher priority than the Macro Queue.
>     - Promises in JavaScript place their resolved/rejected callbacks in the Micro Queue.
>     - After processing the current execution context in the Call Stack, the Event Loop will check the Micro Queue first before the Macro Queue.
> 11. **Web API Queue**:
>     - Web APIs are not part of the JavaScript language itself, but they are built into browsers.
>     - When you use asynchronous actions like `**setTimeout**` or an AJAX request, these are handled not by JavaScript directly but by Web APIs.
>     - Once these actions are completed, their callbacks are placed onto the respective queues (Macro/Micro) to be executed.
> 
> 

> [!note]+ JS Questions
> - What is JS?
> - What is diff between let and var?
> - What is hoisting?
> - What is immutable and mutable?
> - How to call by reference in JS?
> - What is diff between classic and arrow functions and inline functions (self invoking functions)?
> - What is a clone and deep clone?
> - What is a spread operator?
> - Is JS sync or async?
> - What is message queue in JS?
> - Why array in JS is not good enough?
> - How to run a code after some tasks done?
> - == vs === ?
> - When we should use bind()?
> - What is promise?
> - What are the statuses of promise?
> - What is callback hell?
> - Did you write a promise from scratch? (Write and example return value from callback)
> - How to deal with promise, give me an example?
> - What is Promise.all vs Promise.allSettled
> - What is Higher order function/components in js?
> - What is void function and reference functions in JS?
> - Map vs foreach?
> - How to remove a ref of object (infinite levels of objects)?
> - How to handle dates in react native?
> - What is pure functions?
> - What is react native?
> - What is bridging?
> - What is reconciliation?
> - How react-native works?
> - Stateless vs stateful
> - What is JSX?
> - Explain use effect and use state
> - Satte vs props?
> - How to update props?
> - How to pass a ref to a component?
> - What is useImpretiveHandle?
> - What is memoization in react ?
> - How to optimise reactive ? memoization
> - useCallback vs useMemo
> - Lifecycle?
> - What is flex-box?
> - How to monitor API traffic?
> - What is virtual Dom in react?
> - Did you deal with background services ?
> - Did you use realm DB?
> - Which UI library you prefer in react native?
> - What is HTTP handler?
> - Axis or fetch?
> - What is interceptors, or how to pass token with every API?
> - What is props drilling?
> - What is redux?
> - How redux works?
> - How to return function in redux Actions?
> - What is diff between saga and thunk?
> - What is pure component?
> - Why reducer has default case and it is required?
> - Is redux single or multi store?
> - What is reselect?
> - What is redux toolkit?
> - What is context?
> - Redux vs context
> - What is object freeze?
> - What is for - in or how to loop in object?
> - What is typescript?
> - What is the generics in typescript?


> [!note]+ References
> [https://www.freecodecamp.org/news/var-let-and-const-whats-the-difference/](https://www.freecodecamp.org/news/var-let-and-const-whats-the-difference/)

[[JavaScript the hard parts v2]]


System Design

[[System design]]

[[JS Book - Secrets of JS Ninja]]

[[NextJS]]

[[Advance React Book]]

## New

[[JS the hard parts v3]]