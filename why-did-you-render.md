### Why-Did-You-Render

# 如何監控Render的時機點

```ts
function patchFunctionComponent(React, options) {
  const origCreateElement = React.createElement;

/*
// What JSX compiles to behind the scenes
// <Button color="blue" size="large">Click Me</Button>
// becomes:
React.createElement(
  Button,  // The component type (function or class)
  { color: "blue", size: "large" },  // Props object
  "Click Me"  // Children
);

How createElement relates to component rendering
When React processes a functional component:

The component function is passed as the first argument (type) to createElement
React eventually calls this function with the props to get its render output
For hook components, all hook calls execute during this function call
The return value forms part of the virtual DOM tree
--
By intercepting createElement, WDYR can:

See all components about to render
Compare current props with previous props (deep equality check)
Determine if a render was triggered despite equivalent prop values
Report unnecessary renders that could be optimized with React.memo or useMemo
*/
React.createElement = function(type, props, ...children) {
  // For functional components, type is the actual component function
  if (typeof type === 'function') {
    // Now WDYR can wrap this function to monitor:
    // 1. When it's about to render
    // 2. What props it receives
    // 3. Compare with previous render's props
  }
  // ...
}

```

# How does intercept and wrap React components without breaking their original functionality?

# What techniques does createWDYRFunctionalComponentWrapper use to track and preserve component render history across multiple renders

# How does createWDYRFunctionalComponentWrapper implement deep equality comparisons that go beyond React's reference equality checks?

# What criteria does createWDYRFunctionalComponentWrapper use to identify unnecessary renders, and how does it format this information for developers?
