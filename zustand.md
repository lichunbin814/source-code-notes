# zustand

## createStore在做什麼

``` ts
export function createStore(createState) {
  let state
  const listeners = new Set()
  const setState = (partial, replace) => {
    // 設定新狀態，可以直接給物件或用函式取得舊狀態
    state = typeof partial === 'function' ? partial(state) : partial
    // 通知所有監聽者
    listeners.forEach((listener) => listener())
  }
  const getState = () => state // 取得目前狀態
  const subscribe = (listener) => {
    listeners.add(listener) // 加入監聽者
    return () => listeners.delete(listener) // 回傳取消監聽的方法
  }
  // 初始化 state，並把 setState/getState/subscribe 傳給 createState
  state = createState(setState, getState, { setState, getState, subscribe })
  return { setState, getState, subscribe }
}

// example
// 1. 建立 store
const useDogStore = create(() => ({ paw: true, snout: true, fur: true }))

// 2. getState
const paw = useDogStore.getState().paw;

// 3. subscribe
/* 
對應的原始碼
const subscribe = (listener) => {
  listeners.add(listener)
  return () => listeners.delete(listener)
}
*/
const unsub1 = useDogStore.subscribe(console.log)

```
