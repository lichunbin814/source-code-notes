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

