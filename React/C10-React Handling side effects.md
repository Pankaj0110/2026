# Handling Side Effects

From an architectural standpoint, React is designed as a blueprinting engine. It expects components to be pure functions that take props and state and map them deterministically to a UI description.

A side effect is anything that happens outside of this deterministic render cycle. If a behavior touches the "outside world"—such as the browser DOM, a network server, local storage, or a timing subsystem—it is a side effect.

If handled in-correctly, side effects break React’s concurrent rendering engine, cause memory leaks, introduce race conditions, and degrade application performance. Here is an architectural deep-dive into how side effects work and how to manage them at scale.

## 1. The Rendering Pipeline vs. The effect LifeCycle
To manage effects, you must understand the distinction between Render phase and Commit phase.

[ Render Phase ] ────> [ Commit Phase ] ────> [ Passive Effects Phase ]
(Pure, no effects)     (DOM mutations)        (useEffect runs asynchronously)

 - **Render Phase:** React executes your component function to determine the new Virtual DOM tree. This phase must be pure and fast. It can be paused, aborted, or restarted by React (especially in Concurrent Mode). Never trigger side effects here.
 - **Commit Phase:** React synchronously applies changes to the host DOM.
 - **Passive Effects Phase:** After the browser paints the screen, `useEffect` callbacks fire asynchronously. This prevents the effect from blocking user interactions.

**The Architectural Golden Rule**
Do not write side effects directly in the body of your component. > If you fetch data or mutate a global variable directly in the render path, it will execute on every single render trigger, disrupting React's optimization layers.

## 2. Categorizing Side Effects: Event-Driven vs. Synchronization
Architecturally, side effects fall into two distinct buckets. Distinguishing between them prevents over-reliance on useEffect.

**- A. Event-Driven Effects (The Preferred Approach)**
These are triggered by specific user actions (clicks, form submissions, toggles). They belong inside Event Handlers, not useEffect.

*Example*: Sending an analytics ping when a user clicks "Add to Cart".

Why: You know exactly when it happens. It doesn't need to stay synchronized with state; it just needs to fire once upon action.

**- B. Synchronization Effects**
These are triggered because a component rendered and needs to sync its state/props with an external system. These belong in useEffect.

*Example*: Establishing a WebSocket connection based on a roomId prop.

Why: If roomId changes, the old connection must close and a new one must open. It mimics a state machine.

## 3. Deep-Dive: Anatomy of an Architectural Leak
Let's look at a common architectural anti-pattern: a synchronization effect with a severe race condition and memory leak.

**The Anti-Patterns**
```js
// WARN: Flawed Architecture
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`https://api.example.com/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data)); // Race condition & Memory leak hazard!
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

**Why this fails at scale:**
*Race Condition: *If userId changes from 1 to 2, and the request for 2 resolves before the request for 1 finishes, the UI will ultimately overwrite the data with User 1's info when it finally resolves. The UI is now out of sync.

*Memory Leak / Unmounted Component State Update:* If the component unmounts while the fetch is in flight, resolving the promise attempts to call setUser on a non-existent component instance.

**The Architect's Solution: Cleanup & Ignoring**
To fix this, we introduce a cleanup function using an active flag to ignore stale responses.

```js
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let isCurrent = true; // Block scope flag

    async function startFetching() {
      const res = await fetch(`https://api.example.com/users/${userId}`);
      const data = await res.json();
      
      // Only update state if the effect is still valid for this specific userId
      if (isCurrent) {
        setUser(data);
      }
    }

    startFetching();

    // Cleanup function: executes before the next effect run, or on unmount
    return () => {
      isCurrent = false; 
    };
  }, [userId]); // Syncs perfectly whenever userId changes

  return <div>{user?.name}</div>;
}
```
you can also think of abort controller here

## 4. Advanced Patterns for State and Effect Isolation

When building enterprise applications, letting components manage their own raw data-fetching effects leads to a brittle architecture. We use abstraction layers to decouple them.

**A. Escape Hatches: `useEffectEvent`**
Sometimes, you need an effect to read the latest state or prop value, but you don't want that value to re-trigger the effect. Historically, this led to dependency array lying or unnecessary re-runs.

The modern solution (available via experimental/upcoming React features or simulated via refs) is `useEffectEvent`.

```js
import { useEffect, useEffectEvent } from 'react'; // Available in modern/experimental channels

function ChatRoom({ roomId, theme }) {
  // This event handler is non-reactive. It always sees the latest 'theme' 
  // but won't cause the effect to re-run when 'theme' changes.
  const onConnected = useEffectEvent((room) => {
    showNotification(`Connected to ${room}`, theme);
  });

  useEffect(() => {
    const connection = createConnection(roomId);
    
    connection.on('connect', () => {
      onConnected(roomId);
    });

    connection.connect();
    return () => connection.disconnect();
  }, [roomId]); // Look: 'theme' is omitted safely because it's inside useEffectEvent
}
```

**B. Custom Hooks for Synchronization Separation**
Move synchronization logic out of visual components into dedicated custom hooks. This keeps components presentational and makes the effect testable.

```ts
// useIntersectionObserver.ts (Reusable Architectural Utility)
import { useEffect, useState, RefObject } from 'react';

export function useIntersectionObserver(elementRef: RefObject<Element>, options: IntersectionObserverInit) {
  const [isIntersecting, setIsIntersecting] = useState(false);

  useEffect(() => {
    const element = elementRef.current;
    if (!element) return;

    const observer = new IntersectionObserver(([entry]) => {
      setIsIntersecting(entry.isIntersecting);
    }, options);

    observer.observe(element);

    return () => {
      observer.unobserve(element); // Clean up infrastructure
    };
  }, [elementRef, options.root, options.rootMargin, options.threshold]);

  return isIntersecting;
}
```

## 5. Architectural Recommendations for Enterprise Apps
If you are architecting a large-scale React application, follow these structural rules regarding side effects:

1. **Do Not Use `useEffect` for Data Fetching:** Raw useEffect fetching lacks automatic caching, deduplication, background revalidation, and garbage collection. Instead, delegate data-fetching side effects to declarative cache layers like TanStack Query (React Query), SWR, or framework-level solutions (Next.js data loaders, Remix loaders).

2. **Avoid Computed State Effects:** Never use useEffect to transform data when a prop changes.

*Bad:* Reading `props.items`, running useEffect to filter them, and storing them in a local filteredItems state. This triggers an unnecessary second render pass.

*Good:* Transform the data *directly* in the *render phase*, *memoizing it* with `useMemo` if the computation is expensive.

*Strict Mode is Your Guardrail:* In development, React Strict Mode intentionally mounts, unmounts, and remounts components twice. This is designed to instantly surface missing cleanups in your side effects (e.g., duplicate event listeners, unclosed network sockets). If your app breaks under Strict Mode, your side-effect architecture is flawed.


## 6. Notes

- Dealing with side effects, keeping the UI synchronized
  
- Not all side effects need useEffect (mostly only asynchronous jobs need useEffect)
  *example:* if `session storage` is read in application, 'we can do it outside of the **component**'
  Alternative 
  If you only want to read from sessionStorage once when a component mounts, but you want that value to initialize a local state, you pass a function to useState. This is called Lazy Initialization.

  While this function sits inside the component syntax, the function execution is completely decoupled from the standard render path—React runs it exactly once on birth.
  ```ts
  export function ShoppingCart() {
  // React executes this function ONLY on the initial mount.
  // On subsequent re-renders, this function is completely ignored.
  const [cart, setCart] = useState(() => {
    if (typeof window === 'undefined') return [];
    
    const savedCart = sessionStorage.getItem('user_cart');
    return savedCart ? JSON.parse(savedCart) : [];
  });

  return <div>{cart.length} items in cart</div>;
}
  ```

- clearnup function in `useEffect` doesn't execute on first render, it executes on subsiquent renders. because it needs to clean up the effects of the previous render before applying the effects of the new render.
If it ran on the first render, it would destroy the infrastructure you just set up before it even had a chance to do its job.

**Why Running Cleanup on First Render Would Break the Architecture**
If React ran the cleanup function during the first render phase, look at what would happen to our chat room:
Component mounts (roomId = "engineering").
Setup runs: connectToRoom("engineering") opens a WebSocket connection.
If cleanup ran immediately: socket.disconnect() would fire instantly.
Result: The connection is killed milliseconds after it opened. The user sits in a dead chat room because the application immediately cleaned up after itself before any data could be sent or received.

Setup ($Render_1$) $\rightarrow$ Waits for change $\rightarrow$ Cleanup ($Render_1$) $\rightarrow$ Setup ($Render_2$) $\rightarrow$ Waits for unmount $\rightarrow$ Cleanup ($Render_2$).

[State/Prop Changes] ──> [Component Renders] ──> [Browser Paints Screen] 
                                                         │
  ┌──────────────────────────────────────────────────────┘
  ▼
[Run CLEANUP of previous effect] ──> [Run SETUP of new effect]