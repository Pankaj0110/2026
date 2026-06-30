# React Optimization Inbuild techniques

## useCallback

The `useCallback` hook is one of React's primary tools for performance optimization, but it is also one of the most misunderstood. Used correctly, it prevents unnecessary re-renders; used incorrectly, it actually adds minor performance overhead with zero benefit.

### The Core Problem: Referential Equality
To understand `useCallback`, you first need to understand how JavaScript handles object and function comparisons.

In Javascript, functions are objects. Every time a React component re-renders, every function declared inside the component is re-created from scratch on a new memory address.

```js
// On Render 1:
const handleClick = () => console.log('Clicked'); // Memory Address: 0x001

// On Render 2:
const handleClick = () => console.log('Clicked'); // Memory Address: 0x002

console.log(Render1_handleClick === Render2_handleClick); // false
```

While re-creating functions is incredibly fast for JavaScript, the issue arises when that function is passed down as a `prop` to child components. The child component sees a **"new"** `prop` (0x002 instead of 0x001) and triggers a `re-render`, even if the function's logic didn't change at all.


### What `useCallback` Does
`useCallback` solves this by **memoizing** the function instance. It catches the function defination between renders.

```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b], // Dependency array
);
```
- First Render: React creates the function and caches it.
- **Subsequent Renders:** React looks at the dependency array. If a and b haven't changed, React throws away the newly created inline function and returns the exact same memory address from the previous render.


### The Golden Rule: When to Use `useCallback`
You should not wrap every function in `useCallback`. It only provides a performance benefit in specific scenarios.

#### 1. Passing Callbacks to Optimized Child Components `(React.memo)`
This is the most common use case. If a child component is wrapped in React.memo (which prevents re-renders unless props change), passing a non-memoized function completely defeats the purpose of `React.memo`.

```js
import { useState, memo, useCallback } from 'react';

// Child is optimized with memo
const ExpensiveButton = memo(({ onClick }) => {
  console.log('Button rendered!');
  return <button onClick={onClick}>Click Me</button>;
});

export default function ParentComponent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Without useCallback, changing 'text' would cause ExpensiveButton to re-render!
  const handleAction = useCallback(() => {
    console.log('Action triggered');
  }, []); // Empty dependencies = identity never changes

  return (
    <div>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ExpensiveButton onClick={handleAction} />
    </div>
  );
}
```
#### 2. The Function is a Dependency in Other Hooks
If you pass a function into the dependency array of `useEffect`, `useMemo`, or another `useCallback`, omitting useCallback will cause that hook to run on every single render.

```js
const fetchData = useCallback(() => {
  api.get(`/user/${userId}`);
}, [userId]); // Only changes when userId changes

useEffect(() => {
  fetchData();
}, [fetchData]); // Safe: useEffect only runs when userId changes
```
### When NOT to Use It (The Anti-Patterns)

Using useCallback has a cost. React still has to instantiate the inline function on every render and do a shallow comparison of the *dependency* array.

**🚫 Anti-Pattern 1:** Inline functions on normal HTML elements
```js
// BAD: Waste of resources
const handleClick = useCallback(() => {
  console.log('Clicked');
}, []);

return <button onClick={handleClick}>Click</button>;
```
**Why it's bad**: Native HTML tags (like `<button>`) don't care about referential equality; they don't have a "re-render" cycle in the way React components do. You are paying the optimization cost for zero gain.

**🚫 Anti-Pattern 2:** Wrapping a function with massive dependencies
```js
// BAD: Will re-run constantly anyway
const handleState = useCallback(() => {
  logData(a, b, c, d, e);
}, [a, b, c, d, e]);
```
**Why it's bad:** If your dependencies change on almost every render, `useCallback` is doing a bunch of comparison work only to return a new function anyway.

### Pro-Tip: The Functional State Update Trick
A common frustration is having to recreate a callback just because a state variable changed. You can often eliminate dependencies by using the functional updater form of state setters.
**The Sub-optimal Way:**
```js
// Re-created every time 'todos' changes
const addTodo = useCallback((newTodo) => {
  setTodos([...todos, newTodo]);
}, [todos]);
```

**The Optimized way:**
```js
// Identity never changes! No dependencies required.
const addTodo = useCallback((newTodo) => {
  setTodos((prevTodos) => [...prevTodos, newTodo]);
}, []);
```
### Notes
1. Creating a custom hook that returns a function - it is good practice for hook consumers to wrap the returned function using `useCallback` - because if further that is used in a dependency array of `useEffect` in a component then it can cause un-necessary re-renders.

------ // -------------- // --------------------- // ----------------------// -----

## `useMemo`
`useMemo` is not just a syntax-sugar caching mechanism; it is a fundamental primitive in React's reactivity and reconciliation engine designed to control structural identity and optimize the component lifecycle.

To understand `useMemo`, we have to look at how React manages state transitions and fiber nodes.

### **The Fiber Architecture and React's execution phases**
React operates in two main phases:
1. The Render Phase: A declarative phase where React traverses the Fiber tree, calls your component functions, and computes the new virtual DOM. This phase is intended to be pure and side-effect-free.
2. The commit phase: A destructive phase where React mutates the actual DOM to match the new virtual DOM.
3. 
Every time a state change occurs, React triggers a re-render of that component and recursively re-renders all of its children by default. During this render phase, every single line of code inside your component function re-executes.

without `useMemo`, any complex calculation is re-computed on every single tick of the render phase, costing CPU cycles.

#### **1. Inside the HOOK Engine: The Double-Linked List**
Under the hood, React stores a component's hooks as a linked list of "Hook" objects attached to the component's `Fiber node`.

When a component renders for the first time, `useMemo` initializes an object in the linked list. Its inernal structure roughly looks like this.

```js
type MemoHookNode = {
  memoizedState: [any, Array<any> | null]; // [ComputedValue, Dependencies]
  next: Hook | null;
};
```

1. Mount Phase (mountMemo): React executes the factory function, stores the returned value and the dependency array inside memoizedState, and returns the value.

2. Update Phase (updateMemo): React fetches the corresponding hook node from the current Fiber. It performs a shallow comparison (using Object.is) between the old dependencies and the new dependencies.

    - If dependencies are identical: It skips executing the factory function entirely and reads the cached value straight from memoizedState[0].

    - If dependencies differ: It re-runs the factory function, overwrites memoizedState with the new value and new dependencies, and returns the fresh value.

#### **2. Structural Identity and Referential Equality**
While developers often look at useMemo for "heavy computations", its most critical architectural role in large application is maintaining refrential integrity
In JS objects are compared with references, not by value.
```js
{} === {}
[] === []
```

Every time a component re-renders, any object, array, or function declared inline gets a brand-new memory address. This has massive downstream architectural consequences:

*Breaking `React.memo`*
If you pass a non-primitive value (like a configuration object) as a prop to a child component, that child component will always re-render, even if the child is wrapped in `React.memo`. As `React.memo` only performs a shallow prop comparison. If hte object reference changed, the child assumes the data is new.

*Triggering Infinite `useEffect` Loops*
If an object created inline is passed into a `useEffect` dependency array downstream, that effect will run on every single render, potentially triggering infinite state-update loops.

```js
// Architectural Fix using useMemo for Referential Stability
const analyticsPayload = useMemo(() => ({
  userId: user.id,
  timestamp: Date.now(), // assume this handles specific logic
  tier: user.subscription.tier
}), [user.id, user.subscription.tier]); 

// This effect now ONLY runs when the actual data properties change
useEffect(() => {
  api.track('view', analyticsPayload);
}, [analyticsPayload]);
```

#### **3. Advance Use Cases & Patterns**
***A. Memoizing JSX Elements (Alternative to React.memo)***
You don't always need to wrap an entire child component in React.memo to stop it from rendering. You can use useMemo to cache a specific chunk of the Virtual DOM tree.

```js
function Dashboard({ user, updates }) {
  // Expensive Sidebar component only cares about the 'user' object
  const memoizedSidebar = useMemo(() => <Sidebar user={user} />, [user]);

  return (
    <div className="layout">
      {memoizedSidebar} {/* Won't re-render when 'updates' changes */}
      <MainContent updates={updates} />
    </div>
  );
}
```

Why this works: React checks the element reference. If the JSX element reference is exactly the same as the previous render, React completely bails out of rendering that entire subtree, saving reconciliation time.

***B. computational Graphing (cascading Memos)***
In complex data-heavy applications (e.g., dashboards, spreadsheets, or canvas tools), you can chain `useMemo` hooks together to build a highly efficient, reactive computational graph.

```js
const rawData = useDataHook();

// Node 1: Filter raw data (O(N))
const filteredData = useMemo(() => 
  rawData.filter(item => item.isActive), 
[rawData]);

// Node 2: Group filtered data (O(N)) - Only runs if filteredData reference changes
const groupedData = useMemo(() => 
  groupByCategory(filteredData), 
[filteredData]);

// Node 3: Aggregate metrics (O(1) relative to groups)
const metrics = useMemo(() => 
  calculateMetrics(groupedData), 
[groupedData]);
```

If rawData updates but the isActive items remain identical, a properly optimized filter function could return the same reference, cutting off the rest of the computational cascade early.

#### **Cost benifit analysis and Anti-patterns**
Memory isn't free. `useMemo` introduces an architectural trade-off: You are trading memory allocation and dependency-checking overhead for CPU cycles.

**The Overhead of `useMemo`**
Every time you call `useMemo`, React must:
1. Store the dependency array in memory
2. Allocate a function closure for the factory function
3. Loop through the dependencies array on every single render to execute `Object.is` checks.

If you wrap a simple $O(1)$ operation (like const fullMame = useMemo(() => firstName + lastName, [firstName, lastName])), the array allocation and comparison overhead actually make your application slower than just recalculating the string.

**The garbage Trap**

`useMemo` doesn't guarantee that the chached value will never be discarded. React retains the chache as long as the component is mounted.

| Scenario | Use `useMemo`? | Reason |
| :--- | :--- | :--- |
| **Computations running at $O(N^2)$ or worse** | **Yes** | Prevents blocking the main UI thread during render. |
| **Objects/Arrays passed as props to `React.memo` children** | **Yes** | Maintains referential identity to avoid useless child renders. |
| **Objects/Arrays used as dependencies in `useEffect` / `useCallback`** | **Yes** | Prevents breaking downstream dependency comparisons. |
| **Primitive values (strings, booleans, numbers)** | **No** | Values are compared by value anyway; hook overhead outweighs benefits. |
| **Simple $O(1)$ or $O(N)$ operations on small datasets** | **No** | Modern JS engines calculate this faster than React can check dependencies. |


------ // ------------------- // -------------------- // ----------------- // ------


## `**Memo**`

At its core, `React.memo` is a **higher-order component (HOC)** used to optimize performance by preventing unnecessary re-renders of functional components.

By default, a React component re-renders whenever its parent re-renders, regardless of whether its props changed. `React.memo` changes this behavior by shallowly comparing the new props with the old props. If they are identical, React skips rendering the component and reuses the last rendered result.

Interview Tip: Make sure to clarify that `React.memo` is entirely about props change. It does not stop a component from re-rendering if its internal state (useState) or consumed context (useContext) changes.

#### Deep Dive: Under the Hood

##### Shallow comparison & Refrential Equality
By default, React.memo uses a shallow comparison (strict equality === on each prop). This works perfectly for primitive types (strings, numbers, booleans):
```js
// If count goes from 10 to 10, React.memo prevents re-render
<MyComponent count={10} />
```
However, it breaks with objects, arrays, and functions due to JavaScript's referential equality:

```js
// Every time the parent renders, a new object/function reference is created.
// Shallow comparison sees {} !== {}, so React.memo FAILS here.
<MyComponent user={{ name: 'Alice' }} onClick={() => doSomething()} />
```

** The Rescue Crew**: `useMemo`, `useCallback`, and custom comparators
To make React.memo effective when passing objects or functions, candidates must show they know how to pair it with hooks or custom comparison functions.
useCallback to memoize function references.
useMemo to memoize object/array references.
Custom AreEqual Function: React.memo accepts a second argument: React.memo(Component, [areEqual]).

```js
const MyComponent = React.memo(ProductDisplay, (prevProps, nextProps) => {
  // Return true if passing nextProps to render would return the same result as prevProps
  return prevProps.product.id === nextProps.product.id;
});
```

##### Design Patterns Using *React.memo*
**Pattern A:** The pure presentational List item (Leaf component)
The most common and effective patter is memorizing the items in a massive, frequently updating list.

```js
// Without memo, typing in a search bar that filters this list 
// would re-render EVERY ListItem, even unchanged ones.
const ListItem = React.memo(({ item, onSelect }) => {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});
```

**Pattern B:** Splitting State to isolate Expensive Computations

Instead of memoizing everything, a great architectural design pattern is separating components by how frequently their state changes.

If you have a heavy visual component and a fast-updating input component, split them. Pass the heavy one down wrapped in React.memo, or pass it as children (which naturally leverages reference stability if the parent doesn't recreate the children elements).


##### The "Gotchas" & Trade-offs (What Interviewers Actually Care About)

A junior dev wraps everything in React.memo. A senior dev knows that memoization has a cost.

**1. The cost of comparison**
For every single render, React has to loop through the props object and perform an equality check. If a component almost always receives new props (e.g., a wrapper component that accepts `children`), using `React.memo` actually slows down your app because you are doing a useless props comparison plus the inevitable re-render.

**2. Memory Overhead**
React has to store the previous props and the previous rendered JSX tree in memory to perform the comparison. In memory-constrained environments or massive component trees, this can add up.

**3. The "Broken Memoization" chain**
If you wrap a child in `React.memo` but forget to wrap the parent's callback in `useCallback`, your memoization is entirely wasted. You've added comparison overhead for zero performance gain.