# useReducer
A function that reduce one or more complex values to simpler one.

```js
SYNTAX

const [state, dispatch] = useReducer(reducer, initialArg, init?)
```
**reducer:** The reducer function that specifies how the state gets updated. It must be pure, should take the state and action as arguments, and should return the next state. State and action can be of any types.

**initialArg:** The value from which the initial state is calculated. It can be a value of any type. How the initial state is calculated from it depends on the next init argument.

**optional init:** The initializer function that should return the initial state. If it’s not specified, the initial state is set to initialArg. Otherwise, the initial state is set to the result of calling `init(initialArg)`.

It returns an Array

**The current state.** During the first render, it’s set to init(initialArg) or initialArg (if there’s no init).
**The dispatch function** that lets you update the state to a different value and trigger a re-render.

----

If you've ever felt like `useState` gets a bit messy when managing complex state, you are not alone - That is where the `userReducer` hook shines.

Think of `useReducer` as the organized, systematic big brother of `useState`. It is heavily inspired by `Redux`, giving you a structured way to manage state transition based on specific "actions".

## How `useReducer` works (The analogy)

```javascript

const [state, dispatch] = useReducer(reducerFn, initialState)
```
- state: The current state value.
- dispatch: A function you call to trigger a state change (it sends an "action" to the reducer).
- reducerFn: A custom function that calculates the new state based on the action. (***Should be outside the component***)
- initialState: The starting value of your state.
  
### Practical Example: A COUNTER
```js
import React, { useReducer } from 'react';

// 1. Define the initial state
const initialState = { count: 0 };

// 2. Define the reducer function
function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
    default:
      throw new Error(`Unknown action type: ${action.type}`);
  }
}

function Counter() {
  // 3. Initialize useReducer
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <h2>Count: {state.count}</h2>
      {/* 4. Dispatch actions on click */}
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}

export default Counter;
```
### When you should use `useReducer` over `useState`
While `useState` is perfect for independent, simple values(like a toggle switch or text input), you should reach for useReducer when:

- **State depends on other state:** for example, if updating `isLoggedIn` is also require resetting `userData` and setting `isLoading` to false.
- **Complex state logic**: your state is deeply nested object or any array of objects (like a todo list or shopping cart)
- **Next state depends on the previous state:** When the logic gets too complex for functional udeates in `useState`.
- **Easier testing:** Because the `reducer` funtion is a "pure function" (It doesn't rely on react components), you can export it and test it with vanilla Javascript easily.

`useState` vs `useReducer` at Glance

| Feature | `useState` | `useReducer` |
| :--- | :--- | :--- |
| **Complexity** | Best for simple, primitive state. | Best for complex, nested, or related state. |
| **State Updates** | Direct updates (`setCount(5)`). | Dispatched actions (`dispatch({ type: 'set', payload: 5 })`). |
| **Readability** | Great for small components; messy for big ones. | Keeps state logic organized and separated from the UI. |



#### Notes

- React will ignore your update if the next state is equal to the previous state, as determined by an `Object.is` comparison. This usually happens when you change an object or an array in state directly
  