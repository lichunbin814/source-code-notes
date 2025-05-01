# zustand

## createStore在做什麼

``` ts
// 宣告一個名為 `createStore` 的函式，接收一個 `createState` 回呼，用來初始化 state 和 actions
export function createStore(createState) {
// 宣告一個變數 `state`，用來儲存目前的狀態。

  let state

// 建立一個 Set 集合 `listeners`，用來存放所有訂閱 state 變化的監聽函式。

  const listeners = new Set()
  
  //....省略

  // 1.setState
  // 宣告 `setState` 函式，用來更新 state。
  const setState = (partial, replace) => {
// `partial` 可以是物件或函式，`replace` 決定是否完全取代 state。
// 如果 `partial` 是函式，則呼叫它並傳入目前的 state，否則直接使用 `partial` 作為下一個 state。

  const nextState =
      typeof partial === 'function'
        ? partial(state)
        : partial

    // 如果 `nextState` 和目前的 `state` 淺比較（避免沒變化時觸發更新），才繼續。
    if (!Object.is(nextState, state)) {
     // 儲存更新前的 state，方便後續通知 listener。
      const previousState = state
 // 如果 `replace` 為 true 或 `nextState` 不是物件（或為 null），就直接取代 state；否則用淺拷貝合併舊 state 和新 state。
      state =
        (replace ?? (typeof nextState !== 'object' || nextState === null))
          ? nextState
          : Object.assign({}, state, nextState)
// 通知所有 listener，傳入新的 state 和舊的 state。
      listeners.forEach((listener) => listener(state, previousState))
    }
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
對應的關鍵原始碼
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
 對應的關鍵原始碼
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


## 為什麼useSyncExternalStoreWithSelector能自動隨 store 變化而更新
