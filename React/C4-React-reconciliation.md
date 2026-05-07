# React Reconciliation

*React reconciliation is the process React uses to determine:
"Given the `state` or `prop` changed, what is the minimal set of updates needed to make the UI match the new application `state`"

## To understand the reconciliation completely one need to understand below concpts
1. React Elements
2. Virtual Tree
3. Snapshots
4. Diffing Algorithm
5. Fiber Architecture
6. Render vs Commit Phase
7. Scheduling and Priorities
8. Concurrent Rendering
9. Effects and Life cycle handling
10. How updates actually propagate


### 1. The Mental Model
When `state` change
`setCount(count => count + 1)`
React doesn't immediately mutate the DOM

*Instead*
1. React creates a new virtual tree
2. Compares it with previous state
3. Compute differences
4. Applies minimal DOM mutation
**This process is called Reconciliation**

### 2. React Elements - The input to Reconciliation
***Every `JSX` expression becomes a lightweight object***
**Example**
```
  <div className="my-box">Hello</div>
```
**becomes approx.**
```
{
  type: 'div',
  key: null,
  props: {
    className: 'hello-box',
    children: 'Hello'
  }
}
```
These are called `React Elements`
**Important:**
- Elements are immutable
- They are cheap discriptions
- They are not `DOM` nodes
- They are not `Fiber` nodes
**They are simply snapshot of intended UI**

### 3. Virtual DOM (Virtual Tree)
The virtual DOM is essentially a tree of `React Elements`

**Example**
`<APP />`

renders:
```
<div>
  <Header />
  <Content />
</div>
```
React builds a tree:
```
App
 └── div
      ├── Header
      └── Content
```

But react Maintains 2 trees during reconciliation
*|     Tree                    |     Purpose                       |*
| Current Tree                |   UI currently commited to screen |
| Work in progress Tree       |   New Tree being prepared         |

**this dual tree model is fundamental to fiber**

### 4. Snapshots - `React State` as time based snapshot
React treats rendering like taking snapshot in time

Every Render recieve
- A fixed `state` snapshot
- A fixed `props` snapshot

Example:
```
function Counter() {
  const [count, setCount] = useState(0)

  console.log(count)

  return (
    <button onClick={() => {
      setCount(count + 1)
      setCount(count + 1)
    }}>
      {count}
    </button>
  )
}
```
*Both updates see*
`count = 0`

**Because render works from a snapshot** 👈
This is why `React` `batches` updates and process them later

### 5. The Reconciliation problem
Suppose old Tree
```
<ul>
  <li>A</li>
  <li>B</li>
</ul>
```

New Tree:

```
<ul>
  <li>A</li>
  <li>C</li>
</ul>
```
Naively; 
- destroy the whole DOM
- rebuild the entire DOM

**too expensive; React instead performs a heuristic diff.**

### 6. React Diffing algorithm
Theoratically optium tree Diff:
- complexity: O(n3)

Too slow
*React use Heuristic to achieve approximately*
` O(n) `
React Assumpitons:
1. Different Element type produces different trees
2. Keys ideantify stable children

**these assumptions make reconciliation fast*

### 7. Root level comparison
#### *Case 1: - Same type*
**Old**
`<div />`
**New**
`<div />`

React:
- Reuse DOM node
- Update Changes props
- reconciles children

#### *Case 2: Different type*
**Old**
`<div />`
**New**
`<span />`

React destroys entire subtree. *why?:* Because different component types are assumed sematically unrelated.

### 8. Child Reconciliation
This is where key matters

Old:

```
[A, B, C]
```

New:

```
[B, C]
```

React compares by index:
A vs B -> `Replace`
B vs C -> `Replace`
c -> `delete`

which is *BAD*

*With keys*

old:

```
[
  <Item key="A" />,
  <Item key="B" />,
  <Item key="C" />
]
```

New:
```
[
  <Item key="B" />,
  <Item key="C" />
]
```

B → reuse
C → reuse
A → delete

### 9. Real Internal structure - Fiber Nodes
*React used to use a recursive stack based reconciler.*

*Problems*
- non interruptible
- blocks main thread
- poor scheduling
- animation jank (sluggish)

`React fibre` re-wrote the reconciliation completely

### 10. What is Fiber?
Fiber is **A unit of work representing one component instance** 👈

*Fiber is both:*
 - a data structure
 - and execution engine


 ### 11. Fiber Node structure
 A `Fiber` node contains roughly:
 ```
{
  tag, 
  type, 
  key,

  stateNode,

  child,
  sibling,
  return,

  pendingProps,
  memorizedProps,

  memorizedState,

  alternate,

  flags,

  lanes
}
 ```
 - more about it; later

 ### 12. Fiber Tree Relationships
 Fiber uses a `linked list` insead of arrays
 example:
 ```
 App
 ├── Header
 ├── Main
 └── Footer
 ```
 Internally:

 ```
  App.child -> Header
  Header.sibling -> Main
  Main.sibling -> Footer
 ```

 each child has .return
 `.return -> parent`

 **This structure allows Incremental traversal**


 ### 13. Double buffering with alternate
 React maintains:

 |  Tree                  |         Description              |
 |  Current               |       currently visible on UI    |
 | Work in Progress       |       Tree being built           |

 Each fiber has `fiber.alternate`, *Which is linking the two versions* ✅

 This enables:
  - interruption
  - pausing
  - resuming
  - discarding work
  Without corrupting UI.
  *HOW?*
  Suppose React starts rendering a huge update:
  `setItems(largeList)`
  While rendering, the user types into an input box:
  `setSearch('r')`
  The typing update is the high priority.
  Because React is working on the WIP tree, it can stop midway, since the current tree is still intact on the screen, where as the `WIP` tree is partially built.

  Nothing breaks because the visible UI still comes from teh unthouched current tree.
  If react were mutating the current tree directly, interruption would leave the UI in a half-updated inconsistent state.

  - in case of Pausing
    Rendering can pause after some fibers are processed.

    Example:
    ```
    Processed:
    A -> B -> C

    Not yet:
    D -> E -> F
    ```

    The partially completed work stays stored in the WIP fibers.
    Since every fiber has an `alternate`, React already knows:
    - what was previously commited
    - What new work has been completed.
    So react simply freeze the WIP tree temprarily.
    & The current tree continues driving the screen.

  - Resuming
    Later, React resumes from the saved WIP fibers.
    Because the WIP tree already contains completed work:
    ```
    A -> B -> C  ✅ already done
    D -> E -> F  ⏳ continue here
    ```
    React doesn't need to restart from scratch.

    The alternate links help React compare:
    ```
    currentFiber.memoizedProps
    vs
    workInProgress.pendingProps
    ```
    so it knows what still is pending for processing.

    - *Commit phase Swap (the "double buffer swap")*
    When rendering finishes successfully:
    ```
    Current Tree  <-> WIP Tree
    ```

    Example:

    ```
    currentFiber = {
      memoizedState: { count: 1 }
    }

    workInProgressFiber = {
      alternate: currentFiber,
      pendingProps: ...
    }

    ```

    React computes updates on the WIP versions:

    ```workInProgressFiber.memorizedState = { count: 2 }```

    until commit:
    ``` UI still shows count = 1 ```

    After commit:

    ```
    WIP becomes current
    UI now shows count = 2
    ```
    when current become the alternate (WIP) on swap, it just keep that mental model, and it does the future updated in the outdated (current) which is now WIP after swap.


    ### 14. Render phase vs commit phase
    Critical distinction
    ------------------------------------
    Render Phase
    - Also called Reconcillation phase
    - Diffing phase

    React:
      - walks fiber tree
      - Computes updates
      - builds effect list
    Can be:
      - paused
      - interrupted
      - restarted
      - abandoned

  **No Dom mutations accour here**
  **No effects run yet.**

  Commit Phase
  -----------------------------------
  *React applies mutations*
  - DOM updates
  - refs
  - lifecycle effects
  - useEffect scheduling

  Commit phase is synchronous, and can not be interrupted.
  **effects run here, ref updates here** - because here only actual commit is done.

  *There are Sub-steps in commit phase*
  1. *Before mutation phase* 
  React prepare the DOM changes.
  Example:
   `- getSnapshotBeforeUpdate(prevPorps, prevState)` - class lifecycle
   used for reading dom before mutation. Example: scroll Position
  
  2. *Mutation phase*
  Actual `DOM` updates happen.
  ```
  appendChild
  removeChild
  update text
  change attributes
  ```

  UI visible changes.
  Refs may detach temporarily.

  3. *Layout effect phase*
  React runs layout-related effects synchronously.

  Includes:
  ```
  ComponentDidMount
  ComponentDidUpdate
  useLayoutEffect
  ```
  These run before 'Browser' `paint`.
  *Purpose*
  - Measure DOM
  - synchronously adjust layout.

  Example:

  ```
  useLayoutEffect(() => {
    inputRef.current.focus();
  });
  ```

  React guarantees `DOM` already exists.

4. *Passive Effects Scheduling*
Now React schedules passive effects:

`useEffect`

*Important:*
`useEffect` itself usually **does not** run immediately inside the commit phase.
Instead React schedules it to run after paint. - That is why they are called as "passive effects".

*Timeline example*
`
function App() {
  useLayoutEffect(() => {
    console.log("layout");
  });

  useEffect(() => {
    console.log("effect");
  });

  return <div>Hello</div>;
}
`

*ORDER*

```
Render Phase
-------------
build fibers

Commit Phase
-------------
1. DOM updated
2. useLayoutEffect runs

Browser Paint
-------------
screen updates visible

Passive Effects
----------------
useEffect runs

```

*Console*
```
layout
effect
```

*Why `useEffect` is Scheduled*
Because passive effects are usually non-urgent:
- fetch
- analytics
- subscriptions
- logging

React let browser paint first for smooter UI
***👉 Commit phase is performed Atomically, no interruption else UI become inconsistent***

*Life cycle mapping*
- *Class component*
```
| Lifecycle               | Phase           |
| ----------------------- | --------------- |
| render()                | Render          |
| getSnapshotBeforeUpdate | Before Mutation |
| componentDidMount       | Commit          |
| componentDidUpdate      | Commit          |
| componentWillUnmount    | Commit          |

```

- *Function Components*
```
| Hook            | Phase                  |
| --------------- | ---------------------- |
| component body  | Render                 |
| useLayoutEffect | Commit (sync)          |
| useEffect       | After commit (passive) |

```

### 15. Fiber Traversal Algorithm
*Fiber Traversal is depth first*

Pseudo:

```
function performUnitOfWork(fiber) {
  beginWork(fiber)

  if (fiber.child) {
    return fiber.child
  }

  while (fiber) {
    completeWork(fiber)

    if (fiber.sibling) {
      return fiber.sibling
    }

    fiber = fiber.return
  }
}
```
#### beginWork()
This is where reconciliation happens.

React decides:
  - should component re-render?
  - can subtree be re-used?
  - What children should exist?

  For function components:

  `renderWithHooks()`is called.
  This executes hooks
  ```
  useState()
  useMemo()
  useEffect() - does not run here, only records the effec (React store the effect metadata)
  The callback runs later after commit
  ```

  *What `beginWork` decides*
  A. Should component re-render?
  Example: `memo(MyComponent)`
  React may bailout, if `props` & `state` are unchanged. It skips subtree and re-use previous work

  B. Reconciliation
  React compares:
  `old children` vs `new children`
  Example:
  Before
  ```
  <li>A</li>
  **<li>B</li>**
  ```

  After
  ```
  <li>A</li>
  **<li>C</li>**
  ```
  React marks these updates on fibers. Still no `DOM` manupulation.

  C. React create/reuse fibers for children


#### completeWork()
This phase: (also part of render phase)
this happen after children are processed.
  - creates `DOM` nodes
  - prepares updates
  - bubble effects upward

  But still no DOM mutation yet.
  Instead React accumulates effects.
  *Think:*
  ```
  beginWork = go downward
  completeWork = come back upward

  Traversal:
  App
 └── Header
      └── Button

  Order:

  begin App
  begin Header
  begin Button

  complete Button
  complete Header
  complete App  
  ```

  *What Happens in `CompleteWork()`*

  A. Prepare DOM nodes
  for host components:
  `<div>Hello</div>` React may create a DOM notes `React.createElment('div')`
  but it is not attached to the real DOM yet.

  B. Prepare updates
  React computes update payloads

  Example:
  ```
  className changed
  style changed
  text changed
  ```
  stored as effect flags.

  C. Bubble Effects Upward
  Child effects propagate upwards
  ```
  Button needs placement
  Header needs update
  ```
  React bubble these flags upward so parent knows subtree work exists.
  React just collects side effects during render.

  *Both above are part of `Render` phase
  *BIG picture:* React Fiber render work roughly looks like
  ```
  Render Phase
  -------------
  beginWork()
  ```
  execute component
  register useEffect
  reconcile children
  ```
  completeWork()
  ```
  create DOM node
  prepare updates
  mark placement
  ```

  Commit Phase
  -------------
  ```
  beforeMutation
  mutation
  layout effects
  passive effects
  ```

### 16. Effect List
Fibers store side effects:
```
Placement
Update
Deletion
Passive
Layout
```
So react only build linked effects during render
Commit phase later executes them

### 17. Placement/ Update/ Deletion

*Placement*
Insert `DOM` node

*Update*
Modify existing node.

*Deletion*
Remove subtree.

These are stored as flags

### 18. Hooks inside fiber
Hooks are stored on the Fiber node.
Each component Fiber contains linked hook objects.
Example:
```
{
  memoizedState,
  baseState,
  queue,
  next
}
```

Hooks rely entirely on call order
That's why hooks cannot be conditional

### 19. Update Queues
Calling:

`setState(x)`
it doesn't immedietely change state

Instead:
  - create Update object
  - pushes into queue
  - schedule work

  ```
  {
  lane,
  payload,
  next
  }
  ```
### 20. Scheduling and Lanes
React Fiber introduced priority scheduling.

Older React:
`all updates = synchronous`

Fiber:
`updates have priorities`

React uses lanes.

```
| Lane                | Priority   |
| ------------------- | ---------- |
| SyncLane            | urgent     |
| InputContinuousLane | typing     |
| TransitionLane      | non-urgent |
| IdleLane            | background |

```
### 21. Bailouts

Huge optimization, if props/state are unchanged
`oldProps === newProps`
React skips subtree reconciliation.
This is bailout
*used by*
  - memo
  - pureComponents
  - unchanged fibers

### 22. Reconciliation of Arrays
*Core Algorithm*
1. Linear scan
2. Match by key/type
3. Build map for unmatched nodes
4. Reuse/ move/ Delete

This balances speed and correctness

### 23. Why Key matter so much
*Key preserve identity*

Without keys:
`[input, input]`
with keys:
```
<input key="email">
<input key="password">
```
state is preserved correctly.

### 24. State preservation rules
React preserves state when:
- same component type
- same key
- same position

Otherwise state resets.

### 25. Fiber and Suspense
`Suspense` works because Fiber can pause work
React can:
- Render fallback
- Continue hidden tree
- commit later
This require interruptible reconciliation

### 33. Transition Updates
Example:
```
startTransition(() => {
  setSearchQuery(v)
})
```
React mark these updates as low priority, as it is put in `transitionLane`
Urgent updates stay responsive. Enabled by lanes + Fiber scheduler.

### 34. Passive effects
`useEffect` does not run during rendering
Flow:
```
render
commit DOM
paint
run passive effects
```

### 35. Layout effect
`useLayoutEffect` runs:
*after* `DOM` mutation
*before paint*

### 36. Simplified Full lifecycle
*Initial Mount*
```
JSX
 -> React elements
 -> Fiber tree
 -> render phase
 -> complete phase
 -> commit DOM
```

*Update*
```
setState
 -> enqueue update
 -> schedule lane
 -> create workInProgress tree
 -> reconcile
 -> diff
 -> build effects
 -> commit
```

*Internal Reconciliation flow*
High level internals

```
scheduleUpdateOnFiber
  -> ensureRootIsScheduled
  -> performConcurrentWorkOnRoot
  -> renderRootConcurrent
      -> workLoopConcurrent
          -> performUnitOfWork
              -> beginWork
              -> completeWork
  -> commitRoot
```
This is the core Engine.

### 37. All about Reconciliation magic
```
| Concept    | Responsibility   |
| ---------- | ---------------- |
| Render     | calculate UI     |
| Commit     | mutate platform  |
| Scheduler  | prioritize work  | render phase
| Fiber      | store work/state | render phase
| Reconciler | diff trees       |

*simple menteal model*

Update arrives
    ↓
Scheduler prioritizes
    ↓
Render Phase starts
    ↓
Fiber tree processed
(beginWork/completeWork)
    ↓
Effects collected
    ↓
Finished WIP tree
    ↓
Commit Phase
    ↓
DOM mutations + effects

```

## FEW more deep dive topics
Fiber lanes implementation
Hook linked-list internals
Scheduler package internals
Suspense architecture
React Server Components
Selective hydration
Host config & custom renderers
Event priority system
Automatic batching
Offscreen Fiber