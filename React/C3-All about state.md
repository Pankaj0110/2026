# All about React State

React state management has evolved significantly, moving from simple object merging in class components to a sophisticated, **asynchronous scheduling engine**. In `React 19`, *state is less about "variables" and more about "snapshots" and "transitions*."

`useState` is the hook which is frequently used to create a state in the component.

## Best practice - changing the state immutably

if the `state` is an `object` or an   `array` i.e. `reference value`

you should therefore not mutate them directly - *Instead create a deep copy first*

```
const updatedUser = {...user}
updatedUser.name = 'Kite'

if there is an array inside an array (**nested array**)
const updatedStateImmutable = [...prevState.map(arr => [...arr])]

```
### Try to get the resulted state from props, and avoid un-necessary new states

## Before React 18 `<18`
React was only batching updates inside the **event handlers** like *CTA -> `onClick`*. In previous version if you update the state in `setTimeout` or `fetch` callback, or a native `DOM` event, React would trigger a *re-render* every single update.

**i.e. only the eventHandlers state updates were batched**  
```
// Pre-18 behavior in a fetch call
fetch('/api').then(() => {
  setCount(c => c + 1); // Render 1
  setFlag(f => !f);     // Render 2 (Inefficient)
});
```

## React 18 & 19 + **(Automatic batching)**
React now `batches` updates regardless of where they have originated from, whether it's a promise or a timeout, or a native event, React waits for the Micro-task to finish before performing a single render pass.

```
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    // In React 17: This batched (1 render)
    // In React 19: This batches (1 render)
    setCount(c => c + 1);
    setFlag(f => !f);

    setTimeout(() => {
      // In React 17: This caused 2 SEPARATE renders
      // In React 19: This causes only 1 render (Automatic Batching)
      setCount(c => c + 1);
      setFlag(f => !f);
    }, 1000);
  }

  console.log("Rendered!"); // Watch this in your console
  return <button onClick={handleClick}>Update State</button>;
}
```

## Manual Override
There are rare edge cases where you need the DOM to update immediately—for example, if you need to measure the height of a div right after changing its content so you can animate it.

```
import { flushSync } from 'react-dom';

function handleUpdate() {
  // Forces the DOM to update right now
  flushSync(() => {
    setCount(count + 1);
  });
  
  // At this line, the DOM has already been updated.
  // You can now safely measure DOM elements or use third-party DOM libs.
  console.log(document.getElementById('counter').innerText);
}
```
*Warning:* `flushSync` can significantly hurt performance. Use it as a last resort for layout measurements only.

## setTimeout vs promises
### setTimeout sate updates
When you use `setTimeout`, you are telling the browser to run a function in a new *macrotask* after a delay.
```
setTimeout(() => {
  // Execution starts here
  setCount(1); // Update 1 scheduled
  setFlag(true); // Update 2 scheduled
  
  console.log("End of timeout"); 
  
  // NOW the task is finished. 
  // React sees two scheduled updates and batches them into ONE render.
}, 1000);
```
*Does it wait for the execution?* `Yes`. React will not re-render until the code block inside the `setTimeout` has finished running.

### The Promise Case (Microtasks)
Promises work via the *Microtask Queue*. These have higher priority than `setTimeout`.
```
fetch('/api').then(() => {
  // Microtask starts
  setCount(1);
  setFlag(true);
  
  // React waits for this .then() block to complete.
  // Then, before the browser repaints, React performs the single batched render.
});
```


#### The *Batching* internal logic

Think of it like a "Wait and See" approach. When you call a setter function:

  1. React marks the component as "dirty."
  2. React adds the update to a queue.
  3. React checks: "Is there any more code currently running in this same task?"
  4. If No, it triggers the render.
  5. If Yes, it waits until the script finishes, then processes the whole queue at once.

  *Key Distinction: State is a Snapshot*
  Even though React "waits" to render, the value of your state variable does not change inside the current function.
  ```
  const [count, setCount] = useState(0);

  const handleClick = async () => {
    setCount(count + 1);
    console.log(count); // Still 0! 
    
    await someAsyncWork();
    
    setCount(count + 2);
    console.log(count); // Still 0! 
  };

  ```
In the example above, because count is a constant from the current render's "snapshot," it won't reflect the update until the function exits and the component re-renders.

## The Re-render Lifecycle & Child Impact
When state changes, the following happens:

  1. Trigger: setCount is called.
  2. Schedule: React schedules a "render phase."
  3. Render: React calls your function component. It creates a new Virtual DOM tree based on the new state.
  4. Reconciliation (Diffing): React compares the new tree with the old one (using the Fiber architecture).
  5. Commit: React applies only the necessary changes to the real DOM.

### Impact on Children
*By default, if a parent component re-renders, all of its children re-render, regardless of whether their props changed.*
This isn't always a "performance bug"—React is very fast at diffing.
However, if a child is "heavy" (complex calculations or deep trees), this becomes a bottleneck.

## Performance Optimization Strategies
*To prevent unnecessary re-renders, use these patterns:*

  - *Moving State Down:* If only a small part of a tree needs state, don't keep that state in the parent.
  - *Component Composition:* Pass children as props. If Parent re-renders but Child was passed as {children}, React knows the     children prop hasn't changed and skips it. 👍
  - *memo:* Wrap a child component in React.memo to perform a shallow comparison of `props`. *It will only re-render if `props` change*.
  - *useMemo & useCallback:* Use these to maintain referential identity of objects and functions. If you pass a new onClick function on every render to a memoized child, the memoization fails.