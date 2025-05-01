# zustand

## createStore在做什麼

``` ts
export function createStore(createState) {
  let state
  const listeners = new Set()
  
  //....省略

  // 1.setState
  const setState = (partial, replace) => {
    // 設定新狀態，可以直接給物件或用函式取得舊狀態
    state = typeof partial === 'function' ? partial(state) : partial
    // 通知所有監聽者
    listeners.forEach((listener) => listener())
  }

  // 2.getState
  const getState = () => state

  // 3. subscribe
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

把 console.log 當作 listener 傳進去，當 state 有變化時就會被呼叫。
unsub1 是一個取消訂閱的函式。
*/
const unsub1 = useDogStore.subscribe(console.log)

// 1.setState
 /*
 對應的原始碼
const setState = (partial, replace) => {
  state = typeof partial === 'function' ? partial(state) : partial
  listeners.forEach((listener) => listener())
}

這會把 paw 設成 false，然後通知所有 listener（目前只有 console.log）。
*/
useDogStore.setState({ paw: false })

// 取消訂閱，會把 console.log 從 listeners 移除，不再收到通知。
unsub1()

// react hook
function Component() {
  const paw = useDogStore((state) => state.paw)
  ...
}

```
