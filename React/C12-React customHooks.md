# Custom Hooks

In React, **Custom Hooks** are regular javaScript functions, whose name starts with `use` and that can call other hooks. They are the ultimate tool for sharing stateful logic between components without duplicating code or altering you component Hierarchy.

Think of them not as a React feature, but as a natural consequence  of how React Hooks are designed. If you have repetetive logic involving `useState`, `useEffect`, or `useRef`, you extract into a custom hook.

## Advanced Under-the-Hood Principles (Staff/Principle Level)

1. **No shared state, only shared logic**
    A common misconception is that custom hooks share state, They do not.
    Every time a component calls a custom hook, all state and effects inside that hook are isolated. It's a fresh invokation.
2. **The Rule of Hooks & Fiber Node Mechanics**
    Custom hooks rely entirely on the *lexical call order* of hooks inside the component's Fiber Node. React tracks state using a linked list of hook objects on the current Fiber. When a custom hook runs, it is sequentially appends its inner hooks (useState, useEffect) to the same linked list

    **Principal gocha:** Because they flatten into the component's fiber hook list, you cannot call custom hooks conditionally or inside loops. Doing so breaks the linked list index order on subsequent renders, causing catastrophic state mismatches.

3. **Identity Stability & Referential Integrity**
    As a Engineer, you must ensure your custom hooks don't cause accidental re-renders in consumer components. If your hooks returns an object or a function, it should be properly memorized using `useMemo` or `useCallback`
    if It's going to be used in dependency arrays elsewhere.


## Lets Deep dive into Custom hooks using examples
The two classical example of hooks (interview problems) are to implement `useDebounce` and `useThrottle`. It is just to check you mastery of `closures`, `useEffect` cleanups, and `useRef`.

`debounce` and `throttle` both limit how often a function runs, but they do it differently.

**Debounce** wait until the events stop before running it. Good for actions after the user finishes interacting exmpale: typing in an input box (naive exmpale: Lift, it keeps the door open until people are coming in and after a fixed time it close.)
Can delay execution indefinitely if events keep occurring.

**Throttle**: Ensures the function runs at most once every specified interval. Good for actions that should happen continuously but at a controlled rate. exmpale: resize of browser windows
Executes regularly even if events continue.
### useDebounce


```js
import React, { useState, useEffect } from 'react';
import { createRoot } from 'react-dom/client';

const useDebounce = (value, time) => {

  const [state, setState] = useState('');

  useEffect(() => {
    const timer = setTimeout(() =>  setState(value), time);
    return () => clearTimeout(timer);
  }, [time, value]);
  console.log('debounce state:', state)
  return { value: state };
};
const App = () => {
  const [input, setInput] = useState('');

  const handleChange = (e) => { 

    setInput(e.target.value);
  }

  const { value } = useDebounce(input, 3000);

  return (
    <div style={{ padding: '20px' }}>
      <input
        type="text"
        onChange={handleChange}
        value={input}
        style={{ border: '1px solid red', marginTop: '100px' }}
      />
      <div>{value}</div>
    </div>
  );
};

const root = createRoot(document.getElementById('root'));

root.render(<App />);

```

### useDebounce: as a callback

```js
import React, { useRef, useState, useCallback } from 'react';
import { createRoot } from 'react-dom/client';

const useDebounce = (callback, delay) => {
  const timer = useRef()
  return useCallback((text) => {
    if(timer.current) {
      clearTimeout(timer.current)
    }
    timer.current = setTimeout(callback, delay, text)
  }, [callback, delay])
};

const App = () => {
  const [input, setInput] = useState('')
  const [text, setText] = useState('')

  const debounce = useDebounce((text) => {
    console.log('text: ', text)
    setText(text)
  }, 3000)


  const handleChange = (e) => { 
    const text = e.target.value;
    setInput(text)
    debounce(text)
  }

  return (
    <div style={{ padding: '20px' }}>
      <input
        type="text"
        onChange={handleChange}
        value={input}
        style={{ border: '1px solid red', marginTop: '100px' }}
      />
      <div>{text}</div>
    </div>
  );
};

const root = createRoot(document.getElementById('root'));

root.render(<App />);
```
### useThrottle

Simple throttle function can be written as

```js
function throttle(callback, delay) {
  let waiting = false;

  return (...args) => {
    if (waiting) return;

    callback(...args);
    waiting = true;

    setTimeout(() => {
      waiting = false;
    }, delay);
  };
}
```

**React example**

```js
import React, { useState, useEffect, useRef, useCallback } from 'react';
import { createRoot } from 'react-dom/client';
import './index.css';

const useThrottle = (delay) => {
  let waiting = useRef(false);
  const timer = useRef();

  return useCallback(
    (callbackFn) => {
      console.log('waiting', waiting.current);
      if (waiting.current) return;

      callbackFn();
      waiting.current = true;
      clearTimeout(timer?.current);

      timer.current = setTimeout(() => (waiting.current = false), delay);
    },
    [delay]
  );
};

const App = () => {
  const [value, setValue] = useState('');
  const [throttleValue, setThrottleValue] = useState('');

  const throttle = useThrottle(2000);

  const handleChange = ({ target: { value } }) => {
    setValue(value);

    throttle(() => setThrottleValue(value));
  };

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={handleChange}
        style={{ border: '1px solid red' }}
      />
      <p>{throttleValue}</p>
    </>
  );
};

const root = createRoot(document.getElementById('root'));

root.render(<App />);
```



