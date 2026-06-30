# Styling React Application

## There are 5 major ways one can style components

1. Styling with `Vanilla CSS`
2. Scoping styles with `CSS Modules`
3. `CSS in JS` styling with styled components
4. Styling with `tailwind CSS`
5. static and dynamic `conditional` styling


## Let's explain them one by one

1. Styling with `Vanilla CSS`
You can write multiple files with .css extension, then `import` them `import './header.css'`
In results it will Inject multiple css style tags in the `index.html` file
```
<style type='text/css' data-vite-div-id='../path to the file'>
  // css content
</style>
```

| PROS                                    |   CONS                                  |
|-----------------------------------------|-----------------------------------------|
| CSS code is decoupled from 'jsx' code   | Need to know `css`                      |
|You can write code as you prefer         | `css` code is not scoped and css rules  |
|                                         | can clash                               |  

2. Inline Styles
i.e. `<div style={{color: 'red', 'background-color: yellow'}}>content</div>`
- Dynamic or conditional styles
`<div className={valid ? 'valid' : undefined }> content </div>` ✅
`<div className={valid && 'valid'}> content </div>`❌
Because it may result into `false`

- You can add classNames as below
`className={`label $ {emailInvalid ? 'invalid' : ''}`}`

3. Scoping CSS Rules with CSS Modules
* Writing vanilla `css` with file scoping
It is an approch or solution that is implementedby `build process`, it is not a default javascript or browser feature.

It is a process where build tool transform your classNames.
Header.**module**.css, here module is an indicator for build tool to handle it diffrently.

When you create a file named `Button.module.css`, your build tool (webpack/ vite) recognizes the .module suffix. Instead of just injecting the css into the document, the loader treats the file as a `javascript Object`

*Example:*
The source code
```
*Button.module.css*

.error {
  color: red;
}


*Button.jsx*

import styles from './Button.module.css';

function Button() {
  return <button className={styles.error}>Click Me</button>;
}
```

### The transformation under the hood
During the build process, the *`CSS loader`* performs two main tasks

A. Generating the unique Hash
The loader takes the class name (error) and runs it through a hashing algorithm.  This Algo. typically combines 
. The file name (`Button`)
. The orginal class name(`error`)
. A unique has or random string

B. Create a Mapping object
The loader replaces the `imprort` statement in your javascript with a plain object where the keys are your 'human-readable' names and the values are the 'mangled' unique names.

*The transformed JS looks like:*
```
// what styles actually becomes:
const styles = {
  error: "Button_error__z3343x"
}

// The transformed css looks like this
.Button_error__z3343x {
  color: red;
}
```

#### Run time execution
When your react component renders, `styles.error` evaluates to the string `Button_error__z3343x`
1. React injects the string into the DOM as a standard class attribute.
2. The Browser sees `<button class="Button_error__z3343x">`
3. Because the CSS file injected into the `<head>` also uses `.Button_error_z3343x`, the style match perfectly.

**Why it is clever**
- No conflicts: Even if you have another file called Header.module.css with and .error class, it will still be hashed to something else
- Standard CSS: Unlike CSS-in-JS, the final output is still raw .css file, this means you get the performance benifits of browser caching and no runtime overhead for styles calculation.
- Automatic Dead code elimination: If you import `styles` but never use `styles.error`, modern mini-packagers can detect that class isn't need and strip it from the final CSS bundle.


### What if there is `.less`, `.sass` or `.scss` file
Under the hood logic remains the same, but with one extra step in compilation pipeline
  1. The Pre-processor step(`less-loader`): Complies the LESS syntax (vairable, nesting, mixins) into standard "flat" css.
  2. The css Module step(`css-loader`): takes the flat css, generate the unique hash for classnames and create a javascript mapping object
  3. Injection step (`style-loader` or MiniCssExtractPlugin): injects the final, hashed css into the browser.


  👉 The `:global` Escape Hatch:
  Regardless of the extension, if you want a class to stay exactly as you named it(no hashing)
  you use the `:global` prefix. This is common  in LESS/ SASS when stying third party libraries.

4. *Styled components*

`npm i styled-components`

`import { styled } from 'styled-components'`

const div = styled.div` 👈 backtick- tagged template
  display; 'block',
  margin-bottom: '5rem',
  color: ${(props) => {}}
  `

