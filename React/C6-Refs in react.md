# Refs
------------------

*Defination 1*:
When you want a component to 'remember' some information, but you don't want that information to trigger new renders, you can use a `ref`
'Refs' provid a way to access DOM node or react element created in the render method.

*Defination 2*:
In React ecosystem, `Refs` are the escape hatch that allow you to step outside the typical data flow and interact directly with DOM nodes or `persist` values without triggering a `re-render`.

**Use cases**:
- Accessing DOM element with Refs
- Managing values with Ref
- Exporsing API featur from components


## What are Refs?
A `ref` is a plain javascript object with a single property `current`
  - Persistent: The value stays the same between re-renders.
  - Silent: Updating a ref does not trigger a component re-render

*Common use cases*
  - Managing focus, text selection, or media playback
  - Integrating with imperative DOM libraries (like D3 or Google Maps)
  - Storing a value that should not affect the UI


  ## UseRef *vs* CreateRef
  While they look similar, they behave very differentlydue to how React handles component lifecycle.
  - `useRef` (Hooks): Used in functional components, it creates a memorized object that React preserves for the entire lifecycle of the component. You get the same object every time the component re-renders.
  - `createRef` (Legacy/Class): Used in class components. If you use this in a functional component, it will create a brand new ref every time the component renders, effectively losing any value stored in `.current`.

    ## Ref vs state

| ref | state |
|---|---|
| `useRef(initialValue)` | `useState(initialValue)` |
| Returns `{ current: initialValue }` | Returns the current value of the state variable |
| Doesn't trigger re-render | Triggers re-render when you change the value |
| Mutable — you can modify or update the current value outside the rendering process | Immutable — you must change the value using the state setter function, which re-renders the component |
| You **should not read or write the current value** during rendering | You can read state at any time; however, each render has its own snapshot of state, which does not change |



  ## Use cases with exmples

  *A. Accessing the dom focus*
    Most common use is interacting with the elment directly
  ```
  function TextInputWithFocusButton() {
  const inputEl = useRef(null);

  const onButtonClick = () => {
    // "current" points to the mounted text input element
    inputEl.current.focus();
  };

  return (
    <>
      <input ref={inputEl} type="text" />
      <button onClick={onButtonClick}>Focus the input</button>
    </>
  );
}
```
  B. Storing Mutable variables exmple : Timers
  Use refs for the values that change but shouldn't trigger a re-render, like IDs or `setInterval`

  ```
  function Stopwatch() {
  const timerRef = useRef(null);

  const startTimer = () => {
    timerRef.current = setInterval(() => {
      console.log('Tick');
    }, 1000);
  };

  const stopTimer = () => {
    clearInterval(timerRef.current);
  };
}
```


C. Forward Ref
By default, you cannot pass a `ref` prop to a custom component because React treat `ref` as a reserved keyword (like `key`). ForwardRef allows a component to "hand off" its ref to one of its children

```
// The Child:

const MyInput = React.forwardRef((props, ref) => (
  <input ref={ref} className="fancy-input" {...props} />
));

// The Parent

const inputRef = useRef();
<MyInput ref={inputRef} />; // inputRef now points to the internal <input>

```

**Forward ref is no longer required after version 19 React**
```
// React 19+
function MyInput({ props, ref }) {
  return <input {...props} ref={ref} />;
}
```

*D. `useImperitiveHandle`*
Sometimes you don't want to expose the entire DOM node to the parent. Instead, you want to exponse only specific methods. This is used in Conjuction with `forwardRef`

```
const FancyInput = React.forwardRef((props, ref) => {
  const inputRef = useRef();

  **// Only expose a 'focus' and 'scroll' method, not the whole DOM node**
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));

  return <input ref={inputRef} />;
});

```

## Patterns

In industrial-scale React applications, Refs are rarely used for simple "hacks." Instead, they form the backbone of sophisticated patterns that bridge the gap between React's declarative nature and the imperative realities of the browser.

Here are the primary design patterns used in professional software engineering.

### 1. The "Imperative API" Pattern
This is the most advanced pattern, typically used in Design Systems or Component Libraries (like Material UI or Headless UI). By combining forwardRef and useImperativeHandle, developers create a "public API" for a component.

The Goal: Hide internal DOM structure while exposing specific actions (e.g., .open(), .focus(), .validate()).

Industrial Use: A complex Modal component that allows the parent to trigger an "Open" animation without the parent needing to manage the Modal's internal transition state.

### 2. The "Instance Variable" Pattern
React state updates are asynchronous and trigger re-renders. In high-performance applications (like Data Visualization or Trading Dashboards), you often need to store data that changes rapidly but doesn't affect the UI immediately.

The Goal: Avoid "State Lag" or unnecessary re-renders.

Industrial Use: Storing the "previous" value of a prop to compare it with the "current" value, or tracking a "Last Clicked Time" to throttle user actions without causing the entire dashboard to flicker.

### 3. The "Uncontrolled Component" Pattern
While React prefers "Controlled Components" (where state drives every keystroke), this pattern is used in High-Performance Forms (e.g., long surveys with 50+ fields) where keeping every character in state causes typing lag.

The Goal: Improve performance by letting the DOM hold the data, then "pulling" the data once upon submission.

Industrial Use: Massive data-entry tables where using useState for every cell would degrade the frame rate to an unusable level.

### 4. The "Portal & Overlay" Management Pattern
When building Tooltips, Popovers, or Context Menus, you need to calculate the precise pixel coordinates of a "trigger" element to position the overlay correctly.

The Goal: Coordinate-based positioning using getBoundingClientRect().

Industrial Use: Financial charts where hovering over a data point must instantly position a tooltip exactly at that coordinate. The ref provides the "source of truth" for the trigger's location.

### 5. The "External Subscription" Pattern
When integrating React with external logic—like WebSockets, RxJS observables, or Canvas engines—you need a way to keep a reference to the connection object that doesn't disappear when the component re-renders.

The Goal: Maintain a stable reference to a non-React entity.

Industrial Use: Storing a WebSocket instance. If you stored it in state, every message received might trigger a re-render; if you stored it as a regular variable, it would be recreated on every render. useRef keeps it alive and stable.

### Summary: when to rach for these patterns?


**Imperitive API:** you are building a reusable library and need to control a child component "manually"

**Instance Variable:** You need to track data (timer or previous props - without re-rendering)

**Uncontrolled form**: you have hundreds of input and `useState` is making UI feel sluggish

**Overlay/ portal: ** You need to calculate exact screen coordinates for UI elements.

**External Subscription: ** You are bridging react with a legacy JS library or a real-time socket.

## EXAMPLES

### POPUP

```
import { forwardRef, useImperativeHandle, useRef } from 'react'


const Popup = forwardRef((props, ref) => {
  const dialogRef = useRef(null)
  
  useImperativeHandle(ref, () => ({
    open: () => {
      dialogRef.current.showModal()
    },
    close: () => {
      dialogRef.current.close()
    }
  }))
  
  const handleBackdropClick = e => {
    const rect = dialogRef.current.getBoundingClientRect()
    const clickedOnDialog = (rect.top <= e.clientY && e.clientY <= rect.top + rect.height)
     && (rect.left <= e.clientX && e.clientX <= rect.left + rect.width)
    
    if(!clickedOnDialog) {
      console.log("# backdrop clicked")
      dialogRef.current.close()
    }
  }
  return (
    <dialog {...props} ref={dialogRef} onClick={handleBackdropClick}>
      Dialog as a modal
      <form method='dialog'>
        <button>close</button>
      </form>
    </dialog>
  )
})
export default Popup
```

Explanation:
   Dialog position
    {
      top: 100,
      left: 200,
      width: 300,
      height: 150
    }

That means:

dialog starts at x = 200

dialog starts at y = 100

dialog ends at x = 500 (200 + 300)

dialog ends at y = 250 (100 + 150)

e.clientX // horizontal mouse position

e.clientY // vertical mouse position


Is the mouse Y position:
  below the top edge
AND
  above the bottom edge

AND

Is the mouse X position:
  right of the left edge
AND
  left of the right edge


### Timer

