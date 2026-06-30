# React Portals

In React, components usually render neatly inside their parent component's DOM structure. But sometimes, a child component needs to break out of that wrapper visually—like a modal overlay, a tooltip, or a global notification toast.
If a parent container has CSS like `overflow: hidden` or a weird `z-index` stack, your modal might get clipped or hidden entirely.
This is where `React Portals` come in. 
They allow you to "teleport" a component's HTML output to a completely different place in the DOM (like right under the global <body>), while keeping it logically connected to its original parent in your React code.

## How portals work
To use a portal, you use the createPortal function from the react-dom package.

```
import { createPortal } from 'react-dom';

createPortal(children, domNode);
```

 - children: Anything React can render (JSX, components, strings, etc.).
 - domNode: A real browser DOM element where you want this JSX injected (e.g., document.body or an element with a specific ID).

## Practical example of Portal

HTML
```
<!-- index.html -->
<body>
  <div id="root"><!-- Your main React application lives here --></div>
  <div id="modal-root"><!-- Modals will be teleported here --></div>
</body>
```

Here is how you would write the `Modal` component in React using a portal:

```
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay">
      <div className="modal-content">
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.getElementById('modal-root') // The destination
  );
}
```

### The Secret Sauce: Event Bubbling Still Works!
One of the coolest features of React Portals is that they only change the physical location of the DOM element.
They do not change the component's position in the logical React virtual tree.  Because it still sits inside the regular
React hierarchy under the hood:Props & Context: The portaled component can still access Context and hooks from its parent.
**Event Bubbling:** If a user clicks something inside your portal, that click event will bubble up to its `React parent components`, even though those parents are completely separate elements in the browser DOM.**Accessibility Tip:** When teleporting elements like modals or tooltips outside your main tree, you have to be extra mindful of keyboard focus.
Screen readers rely heavily on DOM order, so ensure you manage your focus state (trapping focus inside an open modal) for an accessible user experience!  

Handling keyboard accessibility and focus trapping is the biggest trap developers fall into with modals. Because a portal teleports your modal code directly to document.body or a custom root, screen readers and keyboard users (relying on the Tab key) can accidentally navigate right past the modal and start interacting with the invisible page elements behind it.

An accessible modal needs to do four things right:

  - Focus the modal immediately when it opens.
  - Trap focus inside the modal so pressing Tab wraps around within the modal elements.
  - Close on Escape key press.
  - Restore focus to the exact element (like the button) that opened the modal when it closes.

Here is how you build a production-ready, fully accessible modal component in React using hooks.

```
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

function AccessibleModal({ isOpen, onClose, children }) {
  const modalRef = useRef(null);
  const previousFocusRef = useRef(null);

  useEffect(() => {
    if (isOpen) {
      // 1. Remember what element was focused before opening
      previousFocusRef.current = document.activeElement;

      // 2. Immediately focus the modal container or the first focusable item
      if (modalRef.current) {
        modalRef.current.focus();
      }

      // 3. Listen for keyboard interactions
      const handleKeyDown = (event) => {
        // Escape key to close
        if (event.key === 'Escape') {
          onClose();
          return;
        }

        // Focus trapping strategy for the 'Tab' key
        if (event.key === 'Tab') {
          if (!modalRef.current) return;

          // Find all interactive elements inside the modal
          const focusableElements = modalRef.current.querySelectorAll(
            'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
          );
          
          const firstElement = focusableElements[0];
          const lastElement = focusableElements[focusableElements.length - 1];

          // If no focusable elements, just prevent default tab navigation
          if (focusableElements.length === 0) {
            event.preventDefault();
            return;
          }

          // If Shift + Tab and on the first element, wrap around to the last element
          if (event.shiftKey) {
            if (document.activeElement === firstElement) {
              lastElement.focus();
              event.preventDefault();
            }
          } 
          // If Tab and on the last element, wrap around to the first element
          else {
            if (document.activeElement === lastElement) {
              firstElement.focus();
              event.preventDefault();
            }
          }
        }
      };

      document.addEventListener('keydown', handleKeyDown);

      // Clean up listeners on close
      return () => {
        document.removeEventListener('keydown', handleKeyDown);
        // 4. Return focus back to the button that triggered the modal
        if (previousFocusRef.current) {
          previousFocusRef.current.focus();
        }
      };
    }
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div 
      className="modal-overlay" 
      onClick={onClose} 
      role="dialog" 
      aria-modal="true"
    >
      <div 
        className="modal-content"
        ref={modalRef}
        tabIndex="-1" // Allows container to programmatically receive focus
        onClick={(e) => e.stopPropagation()} // Prevents overlay click from closing
      >
        {children}
        <button className="close-btn" onClick={onClose}>
          Close
        </button>
      </div>
    </div>,
    document.getElementById('modal-root')
  );
}
```

**The ARIA Attributes Matter Too**
Focus trapping handles where the user can navigate, but aria attributes tell the browser's assistive technology what is happening:
**role="dialog":** Informs the browser that this element is a standalone dialog interface separate from the primary page content.
**aria-modal="true":** Tells screen readers that everything else on the page beneath this element is inert/hidden. (For absolute bulletproof setup, you should also apply aria-hidden="true" to your primary #root container while the modal is open).
**tabIndex="-1":** Applying this to the modal content box allows your React code to .focus() it when it opens, meaning a screen reader will immediately start reading the contents of the modal.