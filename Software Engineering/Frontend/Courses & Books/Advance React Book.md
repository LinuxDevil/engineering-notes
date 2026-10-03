# Chapter 1: Intro to re-renders

> [!tip] 💡
> What to expect to learn
> 1. What re-render is and why we need it
> 2. What is the initial source of all re-renders 
> 3. The big re-render myth and why prop changing by themselves doesn’t matter
> 4. Moving state down technique to improve performance
> 5. Why hooks can be dangerous when it comes to re-renders

> [!note] 🔑
> Key takeaways
> 1. Re-renders is how react updates components with new data, without them there will be no interactivity in our apps
> 2. State updates is the initial source of all re-renders
> 3. If a component re-render is triggered, all nested components inside that component will be re-rendered
> 4. During normal react re-render cycles, (<u>With no memoization</u>) prop change doesn’t matter, components will re-render even if they don’t have any props
> 5. We can move the state down to prevent unnecessary re-renders in big apps.
> 6. State update in a hook will trigger the re-render of a component that uses this hook, even if the state itself is not used.
> 7. In the case of hooks using other hooks “custom hooks”, any state update within that chain of hooks will trigger the re-render of a component that uses the very first hook.
> 8. React.memo uses shallow comparison of the props to see if everything was the same or not. 

Examples:

- [https://www.advanced-react.com/examples/01/01](https://www.advanced-react.com/examples/01/01)
- [https://www.advanced-react.com/examples/01/02](https://www.advanced-react.com/examples/01/02)
- [https://www.advanced-react.com/examples/01/03](https://www.advanced-react.com/examples/01/03)

# Chapter 2: Elements, children as props and re-renders

> [!tip] 💡
> What to expect to learn
> 1. How passing components as props can improve the performance of our app
> 2. How exactly react trigger re-renders
> 3. Why components as props are not affected by re-renders
> 4. What an Element is and how it is different than a component
> 5. The basics of react reconciliation and diffing
> 6. What the “children as props” pattern is and how it can prevent re-renders
> 7. “component as props” vs “children as props” pattern

> [!note] 🔑
> Key takeaways
> 1. A Component is a function that accepts arguments (props) and return Elements that should be rendered when this Component renders on the screen
> `const A = () ⇒ <B />` is a Component
> 2. An Element is an object that describes what needs to be rendered on the screen, with the type either a string for DOM elements or a reference to a Component for components 
> `const b = <B />`  is an Element
> 3. Re-renders is just React calling the component function.
> 4. A component re-renders when its element object changes, as determined by `Object.is`  comparison of it <u>before and after re-render</u>
> 5. When elements are passed as props to a component, and this component triggers a re-render through a state update, elements that are passed as props won’t re-render
> 6. “children” are just props and behave like any other prop when they are passed via `JSX` nesting syntax
