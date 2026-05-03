# 1. what is the use of defer attribute in `<script>` tag

- Attribue: none
  Parsing: is paused during download and running the script
  Script download: Immediate
  Script execution: Immediate (block HTML)


- Attribute: deffer
  Parsing: Continous
  Script download: Parallel (background)
  Script execution: After HTML parsing is finished
  Order Preservation: Scripts with defer execute in the exact order they appear in the code.
  DOM Ready: The scripts will *only run after the DOM is fully constructed*, but before the *DOMContentLoaded* **event fires**.

- Attribute: async
  Parsing: Paused only while executing
  Script download: Parallel (in background)
  Execution: As soon as download finish

# When should you use it?
When your script needs the DOM: If your code tries to grab an element (like document.getElementById('btn')), defer ensures that element actually exists before the code runs.

For dependent scripts: If Script B relies on Script A, using defer on both ensures A always runs before B.

Performance: It is generally the best practice for most external scripts because it prevents the "white screen" effect where users wait for scripts to load before seeing the page content.

# 2. What is `<noscript> tag`?
The `<noscript>` tag is a fallback mechanism used to display alternative content for users who have disabled JavaScript in their browser or are using a browser that doesn’t support it.

Think of it as a "Plan B." If JavaScript is running, the browser completely ignores everything inside the `<noscript>` tags. If JavaScript is off, the browser reveals that content to the user.

  ## Common Use Cases
  Warning Messages: Informing users that the website requires JavaScript to function correctly.

  Alternative Links: Providing a link to a basic, non-JS version of the site.

  Tracking & Analytics: Many services (like Google Tag Manager or Facebook Pixel) use a `<noscript>` tag to fire a tracking pixel so they can still gather basic data even if the main script is blocked.

  CSS Fallbacks: Applying specific styles to ensure the page remains readable without interactive elements.

# 3. Why react application needs a build process?
  React requires a build process because the code developers love to write is not the code browsers actually understand. When you build a React app, you are essentially translating and optimizing your source code into a high-performance package that can run on any user's device.
  1. JSX Transformation
    React uses JSX (JavaScript XML), which allows you to write HTML-like structures directly inside your JavaScript. However, browsers can only read standard JavaScript objects and strings.

   The Problem: Browsers see `<div>Hello</div>` in a JS file and throw a syntax error.

   The Build Solution: Tools like Babel transform that JSX into standard function calls like `React.createElement('div', null, 'Hello')`.

  2. Modern JavaScript (ES6+) Support
    Developers use modern features like optional chaining, nullish coalescing, and arrow functions.
 Older browsers (or even some current ones) might not support the latest ECMAScript standards.

The Build Solution: The build process "transpiles" your modern code into older versions of JavaScript (like ES5) to ensure your app doesn't crash for users on older browsers.

  3. Module Bundling
    A typical React project has dozens, if not hundreds, of separate .js, .css, and image files. If a browser tried to download each file individually, the site would be incredibly slow.

   The Build Solution: Tools like Webpack, Vite, or Rollup "bundle" all those separate files into one or two small files. This reduces the number of HTTP requests the browser has to make.

  4. Performance Optimization
    During development, your code is written for readability. During production, the code needs to be optimized for speed. The build process performs:

   Minification: Removing all whitespace, comments, and shortening variable names to shrink file size.

   Dead Code Elimination (Tree Shaking): Removing code or libraries that you imported but never actually used.

   Image Compression: Automatically shrinking asset sizes.


  # 4. What is tree shaking and Rollup in React build process?
   Tree Shaking is a term for dead-code elimination. Imagine your code and all its dependencies as a giant tree. The functions you actually use are the "living" branches. Tree shaking "shakes" the tree so that the "dead leaves" (the functions you imported but never used) fall off, leaving only the essential code in your final bundle.

   ## How it works:
   It relies on the ES6 Module syntax (import and export). Because import statements are static (they happen at the top of the file and don't change at runtime), the build tool can determine exactly which pieces of a library are being used before the code even runs.

Example: You import a massive math library just to use a squareRoot function. Tree shaking ensures the other 500 math functions don't end up in your user's browser.

   ## What is Rollup?
   Rollup is a modern JavaScript module bundler. While Webpack was traditionally the king of bundling (and is still widely used), Rollup became famous for being the first to truly master tree shaking and producing very "lean" bundles.

   ### Why Rollup is used in React:
   Efficiency: Rollup doesn't add a lot of extra "glue code" to your files. It stitches your modules   together in a way that is very close to standard JavaScript, making the final file smaller and faster to execute.

   Library Focused: Most React libraries (like Framer Motion or Redux) are bundled using Rollup because it creates clean, reusable packages.

   Vite Power: If you use Vite (the most popular modern way to start a React project), it actually uses Rollup under the hood for its production build process.
  ### How they work together in the Build Process
   When you run a command like npm run build, the process generally looks like this:

 - Discovery: Rollup starts at your entry point (usually index.js).

 - Analysis: It follows every import statement to map out every file in your project.

 - Tree Shaking: As it builds the map, it flags any exported functions that are never called.

 - Bundling: Rollup merges all the "alive" code into a single file (or a few "chunks"), stripping    out      the unused code.

 Output: You get a highly optimized .js file ready for the browse