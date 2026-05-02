
# Define react.js, react-dom.js and react-scripts.js?
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


# How React handles a component and how it builds a *Component Tree* ?


