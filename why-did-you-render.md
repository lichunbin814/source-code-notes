### Why-Did-You-Render

# 如何監控Render的時機點

```ts
function patchClassComponents(React, options = {}) {
  const origRender = React.Component.prototype.render
  
  React.Component.prototype.render = function () {
    // Track component before render
    trackRenderInfo(this)
    
    // Call original render
    const result = origRender.apply(this, arguments)
    
    // Compare with previous render and notify about unnecessary renders
    notifyIfUnnecessaryRender(this)
    
    return result
  }
}
```

# 判斷不必要渲染的核心邏輯
```ts
function isRenderUnnecessary(prevProps, nextProps, prevState, nextState) {
  // Deep equal comparison to check if props changed substantively
  const arePropsEqual = deepEqual(prevProps, nextProps)
  const isStateEqual = deepEqual(prevState, nextState)
  
  // If both props and state are deeply equal but component still rendered
  return arePropsEqual && isStateEqual
}
```

# deepEqual在做什麼
```ts
function deepEqual(a, b, circularReferences = new WeakMap()) {
  // Handle primitive types
  if (a === b) return true
  
  // Different types or null values
  if (!a || !b || typeof a !== 'object' || typeof b !== 'object') {
    return false
  }
  
  // Handle circular references
  if (circularReferences.get(a) === b) return true
  circularReferences.set(a, b)
  
  // Compare object keys and values recursively
  // ...
}
```

# 怎麼比較function是否相同
```ts
function areFunctionsEqual(prevFunc, nextFunc, options) {
  // Functions are never equal by reference in JS unless they're the same instance
  if (prevFunc === nextFunc) return true
  
  // Check if functions have the same string representation (limited approach)
  if (options.compareFunctionsBy === 'name') {
    return prevFunc.name === nextFunc.name
  }
  
  if (options.compareFunctionsBy === 'toString') {
    return prevFunc.toString() === nextFunc.toString()
  }
  
  return false
}
```
# 重新渲染的原因是什麼?

```ts
function getUpdateInfo(prevProps, nextProps, prevState, nextState) {
  // Find which props changed
  const changedProps = getChangedProps(prevProps, nextProps)
  
  // Find which state keys changed
  const changedState = getChangedProps(prevState, nextState)
  
  // Determine if render was caused by props or state
  const reason = Object.keys(changedProps).length ? 'props' : 
                (Object.keys(changedState).length ? 'state' : 'unknown')
  
  return {changedProps, changedState, reason}
}
```
