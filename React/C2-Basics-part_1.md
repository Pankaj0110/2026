
# 1. Define react.js, react-dom.js and react-scripts.js?
  ## 1. react (The Brains)
  The react package is the core library. It contains only the logic necessary to define components and handle the Virtual DOM.

 ### What it does:
It provides the "rules" for how components work. It includes features like useState, useEffect, and the logic for creating elements.

 ### Platform Independent:
Interestingly, the react package doesn't know anything about the "web." It just knows how to manage state and component hierarchies. This same package is used for React Native (mobile) and React VR.

  ## 2. react-dom (The Glue)
  If react is the brain, react-dom is the hands. It is the package that actually talks to the web browser.

### What it does:
It takes the Virtual DOM created by the react package and "renders" it into the actual HTML DOM that the user sees.

 ### The Bridge:
It handles all the browser-specific logic, like efficiently updating an input field or handling a click event.

 Main Method: You’ll typically see it used once in your index.js file: root.render(<App/>).

  ## 3. react-scripts (The Toolbox)
  This package is specific to Create React App (CRA). It is a set of scripts and configurations that run in the background on your computer (not in the user's browser).

  ### What it does:
It manages the complex Build Process we discussed earlier. It hides all the complicated configurations for *Babel (translation), Webpack (bundling), and ESLint (error checking)*.

    Key Commands:

    npm start: Launches a local development server.

    npm run build: Uses Rollup/Webpack to bundle your app for production.

    npm test: Runs your test suites.


# 2. How React handles a component and how it builds a *Component Tree* ?

`ReactDom.createRoot(elm).render(<App />)`

* Built in components * 
- Name starts with lower case
- They are valid and officailly defined HTML elements
- They are rendered as DOM nodes by React (displayed on screen)

* Custom Components *
- Name starts with lower case
- Defined by you, and return a valid HTML elment, may wrap multiple elements under one Container `tag`.
- React traverses the component tree until it left with only the `html` elmements.


# 3. Are there any alternative prop syntax?
Usually you want ``` <MyComponent className="myClass" {...rest} /> ``` if className is also existing inside the `...rest` object, then the `className="myClass"` is overridden ❌.

👉 ``` <MyComponent {...rest} className="myClass"> ``` This is the right way ✅

in the events , it bydefault pass the `event` object


# 4. What are React Fragments?
JSX should have one parent element to return, simillarly as of funcions
So in this case `<>` or `<Fragment>` helps.

** If we don't have to use any class etc on parent, `Fragment` is the best choise in this case. **

# 5. Brief about keeping an image in Public folder vs Asset/ inside src.

## Storing images in public folder
You can store the images in public folder and then directly refernce them from inside of your index.html or index.css file.
"Reson for that is files stroed in public folder are made publicly availiable by underlying project development server & **Build process**". Just like index.html file, those files are also available from browser & can be accessed or request by other
files

## Storing image under src/ directory
Any file under `src` folder is not made public, they can't be accessed by website visitors like the filed in public folder.
example: **if you load http://localhost:3000/src/assets/logo.png** - it will throw an error.

Instead files stored under the `src/` can be used in your code files.  Image imported to the code files are then picked up by the underlying build process. Potentially optimized and kind of *injected* **into** `public/`, right **before serving the website**.
Links to these files are automatically generated & used in the places where you referenced the imported files.