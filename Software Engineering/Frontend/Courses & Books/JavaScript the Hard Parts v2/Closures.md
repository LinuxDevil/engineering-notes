![[Screenshot_2023-08-29_at_10.27.33_PM.png]]

Lexical Scoping

Closures
    A function that holds a reference to the outer state.
    A function that has a memory 

Memoizations
    A go to practice to enable us to not repeat calculations and tasks that has been run before.


![[Screenshot_2023-08-30_at_11.11.04_PM.png]]

```typescript
// ------------------------------
// Hoisting
// ------------------------------
// createFunction: <function> (The whole function code)
// generatedFunc: *uninitialized*
// result: *uninitialized*

// ------------------------------
// Global Memory 
// ------------------------------
// createFunction: Function  (The whole function code)
// generatedFunc: undefined
// result: undefined

// ------------------------------
// Call Stack
// ------------------------------
// Global
// createFunction()
// generatedFunc(3)
// multiplyBy2(3)

// ------------------------------
// Global Memory
// ------------------------------
// createFunction: Function  (The whole function code)
// generatedFunc: multiplyBy2  (The whole function code)
// result: 6
```

Another Example

```typescript
function outer() {
    let counter = 0;
    function incrementCounter() {
        counter++;
        console.log('increment-counter: counter', counter);
    }
    incrementCounter();
    console.log('outer: counter', counter);
}

outer();
```

Under the hood

```typescript
// Global Memory
outer: <function> (The whole function definition)

// ====== outer() =======
// Execution Context
Execution ========== Local Memory
incrementCounter()		counter: 0
											incrementCounter: <function> (The whole function definition)
		// ====== incrementCounter() =======
		// Execution Context
		Execution ========== Local Memory
			counter++ (This will lookup for the local memory of the incrementCounter)
								(Then it will go to the upper scope and look it up, it will find it)

```

![[Screenshot_2023-08-31_at_8.27.13_PM.png]]

```typescript
===Global Memory===
outer: <function> (The whole function code)
myNewFunction: *uninitialized*
	|===Call Stack===
	|global
	|outer()
	|	===== Outer Exection Context ====
	|	=== Execution === === Local Memory ===
	|											counter: 0
	|											incrementCounter: <function> 
	|				              (This will get a hidden property [[scope]] that will reference the 
	|						           the scope )
	|		-> returned the value of the incrementCounter
	|				as the value of myNewFunction
		____________________________________________
myNewFunction: <function> the definition 
								with all it's scope (the counter variable)
								called *live data* { counter: 0 }
	|===Call Stack===
	|global
	|myNewFunction()

	|===== myNewFunction Exection Context ====
	|=== Execution === === Local Memory (Variable Enviornment) ===
	|		counter++    | (This will be looked at the local memory)
	|								 | (Then it will look in the saved scope "The packback" )
	|								 | (The packback is a hidden variable called scope)
	|								 | (Then increment it)
	____________________________________________
myNewFunction: <function> the definition 
								with all it's scope (the counter variable)
								called *live data* { counter: 1 }

	|===Call Stack===
	|global
		--------------

	|===Call Stack===
	|global
	|myNewFunction()
		-------------

	|===== myNewFunction Exection Context ====
	|=== Execution === === Local Memory ===
	|		counter++    | (This will be looked at the local memory)
	|								 | (Then it will look in the saved scope "The packback" )
	|								 | (The packback is a hidden variable called scope)
	|								 | (Then increment it)
		__________________________________________________
myNewFunction: <function> the definition 
								with all it's scope (the counter variable)
								called *live data* { counter: 2 }
	|===Call Stack===
	|global
	 -------------
```

Scope is just really what variable (data) is available for me in that exact line that I’m running.

JS Scope rule is **lexical/static** scoping: Where i save my function determines for the rest of that function life what data it will have access to. (Not where you run it That is Dynamic Scoping) so even if you returned the function you will grab the data and save it.

They refer to the “backpack” as (Closed Over Variable Environment C.O.V.E) or (<u>Persisted Lexical/Static Scoped Referenced Data</u>) then they just named it a **Closure**.

The backpack is the result of lexical scoping.

Memoization → Caching of a slow task 


