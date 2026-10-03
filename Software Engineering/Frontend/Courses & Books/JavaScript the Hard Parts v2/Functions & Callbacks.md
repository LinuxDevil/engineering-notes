When we create function we need to think about their reusability and not to break the DRY principle.

Bad function:

![[Screenshot_2023-08-26_at_7.10.39_PM.png]]

```typescript
------ Global Memory ------
copyArrayAndMultiplyBy2: <function>
myArray: [1,2,3]
result: *uninitialized*
				--- Create a new execution context ---
					result = copyArrayMultiply2([1,2,3])
									-------- New Execution Context --------
						-------------------Memory------------Call Stack----
													  |array: [1,2,3]    |[global, copyArrayMultiply2]
													  |output: []				 |
				array: [1,2,3]	    |								 	 |
				1 * 2 pushed to     |									 |
				output: [2,  ]	    |									 |
			  2 * 2 pushed to     |									 |
			  output: [2,4, ]     |									 |
			  3 * 2 pushed to     |									 |
			  output: [2,4,6]     |									 |
			  lookup the value of |									 | 
			  output and return it|									 |
				to the result       |                  |[global]
									-------- End of Execution Context --------
result: [2,4,6]
```

Good function:

Now by passing the function as a parameter, leaving out some of our functionality undetermined:

![[Screenshot_2023-08-27_at_10.52.01_PM.png]]

```typescript
------ Global Memory ------
copyArrayAndManipulate: <function>
multiplyBy2: <function>
result: *uninitialized*
				--- Create a new execution context ---
					result = copyArrayAndManipulate([1,2,3], multiplyBy2)
								-------- New Execution Context --------
					-------------------Memory------------Call Stack----
												  |array: [1,2,3]    |[global,copyArrayAndManipulate]
													|instruction:      |[global,copyArrayAndManipulate,multiplyBy2]
													|      <function>  |
												  |output: []				 |
			array: [1,2,3]	    |								 	 |
			calling instructions|
										*Creating a new execution context*
															----Memory---
																input: 1
														return the value 
		  and get returned value to be 
			pushed to output    |									 |global,copyArrayAndManipulate]
			output: [2,  ]	    |									 |
		  calling instructions| 	   	   	   	   |
										*Creating a new execution context*
															----Memory---
																input: 1
														return the value 
		  pushed to    	   	  |									 |global,copyArrayAndManipulate,multiplyBy2]
		  output: [2,4, ]     |									 |global,copyArrayAndManipulate]
		  calling instructions| 	   	   	   	   |
										*Creating a new execution context*
															----Memory---
																input: 1
														return the value 
		  pushed to    	   	  |									 |global,copyArrayAndManipulate,multiplyBy2]
			output: [2,4,6]     |									 |global,copyArrayAndManipulate]
		  lookup the value of |									 | 
		  output and return it|									 |
			to the result       |                  |[global]
								-------- End of Execution Context --------
result: [2,4,6]
```

How could we pass the function as a parameter ?

![[Screenshot_2023-08-27_at_11.04.24_PM.png]]

First class means everything that an object have it have it too, it can be treated like objects.

behind the scenes functions are just an objects.

A higher order function: any function that takes or returns a function.

HoF and Callbacks keep our code DRY.