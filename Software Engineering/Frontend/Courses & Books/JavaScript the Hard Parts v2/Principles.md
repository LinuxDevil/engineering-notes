---
title: "Principles"
---
![Screenshot_2023-08-26_at_6.03.43_PM](/Attachments/Screenshot_2023-08-26_at_6.03.43_PM.png)

Thread of execution → <u>The ability to go through the code and execute it</u>

Going code line by line and saving data in the memory If you combine going the code line by line and having a space of memory is called <u>**execution context**</u>

```typescript
----------------- Memory -----------------
num: 3
multiplyBy2:  <Function>
output: *uninitialized*
	--> 
		output = multiplyBy2(3)
					create a new execution context
					------------Local Memory---
									 |  inputNumber: 3 *parameter*: *argument*
									 |  result: 6
									 |  
			go lookup the| 
			result and   |
			take it      |
			(return it) as| 
			the output value|
       \
output: 6
newOutput: *uninitialized*
newOutput: multiply2(10);
					create a new execution context
					------------Local Memory---
									 |  inputNumber: 10 *parameter*: *argument*
									 |  result: 20
									 |  
			go lookup the| 
			result and   |
			take it      |
			(return it) as| 
			the output value|
		       \
newOutput: 20
---------------------------------------------------
```

CallStack:

```typescript
When we call the multiplyBy2(3) it gets pushed to the call stack

|              |
|multiplyBy2(3)| 
|____Global()__|

When we hit return it is poped up

|              |
|              | 
|____Global()__|

When we hit the multiplyBy2(10) again

|               |
|multiplyBy2(10)| 
|____Global()___|

Then when we return it is poped up

|              |
|              | 
|____Global()__|
```

