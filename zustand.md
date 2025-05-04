# zustand

## create裡會做什麼事情

```ts
import { create } from 'zustand'

const useStore = create((set) => ({
  bears: 0,
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
   ...
}))

// 實際上會執行
// create 的原始碼（src/react.ts）
export const create = (createState) =>
  createState ? createImpl(createState) : createImpl

function createImpl(createState) {
  // 建立一個 store API，包含 getState、setState、subscribe 等方法
  const api = createStore(createState)

  // 建立一個 React hook（useBoundStore），用來在組件中訂閱 store 狀態
  // 例: 這裡 state => state.count，其實就是api.getState().count
  const useBoundStore = (selector) => useStore(api, selector)

  // 把 store API（如 getState、setState、subscribe）合併到 hook 上
  /*
 可以這樣用：
useBoundStore.getState() // 取得 state
useBoundStore.setState({ count: 2 }) // 設定 state
   */
  Object.assign(useBoundStore, api)

  // 回傳這個 hook，這個 hook 物件同時有 getState、setState、subscribe 等 API，可以在 component 外直接操作 store
  return useBoundStore
}


create 幫你包裝好 store，讓你在 React 裡用 hook 方式取得和操作 state。
內部其實就是呼叫 createStore 產生 store，再用 React hook 包起來。

create((set) => {...})
   │
   └─> createImpl(createState)
           │
           └─> createStore(createState)
                   │
                   └─> 建立 state、setState、getState、subscribe
           │
           └─> 建立 React hook（useStore）
           │
           └─> 合併 API
   │
   └─> 回傳 useStore（可在 React component 內使用）

```
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

/*
example
// partial 是 function
store.setState(state => ({ count: state.count + 1 }))
// 這裡 partial 是 (state) => ({ count: state.count + 1 })

// partial 不是 function（是物件）
store.setState({ count: 100 })
// 這裡 partial 是 { count: 100 }

*/
  const nextState =
      typeof partial === 'function'
        ? partial(state)
        : partial

    // 如果 `nextState` 和目前的 `state` 淺比較（避免沒變化時觸發更新），才繼續。
    if (!Object.is(nextState, state)) {
     // 儲存更新前的 state，方便後續通知 listener。
      const previousState = state
 // 如果 `replace` 為 true 或 `nextState` 不是物件（或為 null），就直接取代 state；否則用淺拷貝合併舊 state 和新 state。
/*
example

假設原本 state 是：
{ a: 1, b: 2, c: 3 }

// replace 為 false（預設），會合併 state
store.setState({ a: 100 }, false)
// 結果：{ a: 100, b: 2, c: 3 }

// replace 為 true，會完全取代 state
store.setState({ a: 100 }, true)
// 結果：{ a: 100 }
*/
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

## useSyncExternalStoreWithSelector是做什麼的
用一個最簡單的自訂 store，搭配 useSyncExternalStore，從初始化、監聽到通知，完整解釋一次：

1. 初始化 store

``` ts
let state = 0
const listeners = new Set<() => void>()

function getSnapshot() {
  return state
}

function subscribe(listener: () => void) {
  listeners.add(listener)
  // 回傳一個取消訂閱的函式
  return () => listeners.delete(listener)
}

function setState(newState: number) {
  state = newState
  // 通知所有 listener
  listeners.forEach(listener => listener())
}
```
3. 在 React 組件中使用
```ts
import { useSyncExternalStore } from 'react'

function Counter() {
  const value = useSyncExternalStore(subscribe, getSnapshot)
  return (
    <div>
      <span>{value}</span>
      <button onClick={() => setState(value + 1)}>+1</button>
    </div>
  )
}
```
4. Timeline 詳細說明

(A) 組件初次渲染
React 執行 getSnapshot()，取得 state（例如 0）。

React 執行 subscribe(listener)，把一個內部 listener 加到 listeners Set 裡。

(B) 你點擊按鈕，呼叫 setState

setState(value + 1) 改變 state（例如變成 1）。

執行 listeners.forEach(listener => listener())，通知所有 listener。

React 的 listener 被呼叫，React 重新 render 組件。

React 再次執行 getSnapshot()，取得最新 state（1），畫面更新。

(C) 組件卸載

React 會呼叫 subscribe 回傳的取消訂閱函式，把 listener 從 listeners Set 移除。

5. 重點整理
初始化：你自己寫一個 state 和 listeners 集合。
監聽：subscribe 讓 React 把自己的 callback 加進 listeners。
通知：setState 改變 state 後，呼叫所有 listeners，讓 React 知道要重新 render。


## 為什麼useSyncExternalStoreWithSelector能自動隨 store 變化而更新

## useSyncExternalStore的實現邏輯

```ts
1. 初始渲染階段
當組件首次渲染時，useSyncExternalStore 執行以下步驟：

## 獲取初始快照：
const value = getSnapshot();

在每次渲染時，getSnapshot 會被調用以獲取當前的狀態快照（例如 Redux store 的狀態）。這是同步執行的，確保組件在渲染時使用最新的快照。

## 初始化 useState：
const [{inst}, forceUpdate] = useState({inst: {value, getSnapshot}});

這裡使用 useState 來儲存一個物件 {inst: {value, getSnapshot}}，並獲取 forceUpdate 函數用於強制重新渲染。inst 是一個可變物件，用來追蹤快照和 getSnapshot 函數。

## 檢查快照一致性：
if (__DEV__) {
  if (!didWarnUncachedGetSnapshot) {
    if (value !== getSnapshot()) {
      console.error('The result of getSnapshot should be cached to avoid an infinite loop');
      didWarnUncachedGetSnapshot = true;
    }
  }
}
在開發模式下，React 會檢查 getSnapshot 的結果是否穩定。如果連續調用 getSnapshot 返回不同的值，會警告可能導致無限迴圈。這強調 getSnapshot 應返回穩定的值（例如 Redux 的 store.getState()）。



2. useLayoutEffect 階段
在渲染完成後，React 會執行 useLayoutEffect，用於更新 inst 物件並檢查快照是否變化：
useLayoutEffect(() => {
  inst.value = value;
  inst.getSnapshot = getSnapshot;

  if (checkIfSnapshotChanged(inst)) {
    forceUpdate({inst});
  }
}, [subscribe, value, getSnapshot]);
## 更新 inst：
inst.value = value：將當前快照值儲存到 inst.value。
inst.getSnapshot = getSnapshot：更新 getSnapshot 函數引用。

## 檢查快照變化：
checkIfSnapshotChanged(inst) 會比較 inst.value（上一次的快照）與當前 getSnapshot() 的結果：
function checkIfSnapshotChanged(inst) {
  const latestGetSnapshot = inst.getSnapshot;
  const prevValue = inst.value;
  try {
    const nextValue = latestGetSnapshot();
    return !is(prevValue, nextValue);
  } catch (error) {
    return true;
  }
}
如果快照變化（prevValue !== nextValue），則調用 forceUpdate({inst})，這會觸發組件重新渲染。
## 觸發時機：
useLayoutEffect 在 DOM 更新後但瀏覽器繪製前執行。如果在這階段檢測到快照變化，會立即觸發重新渲染，確保組件使用最新的快照。

3.useEffect 階段（訂閱 store）
在 useLayoutEffect 之後，React 執行 useEffect，這裡是 subscribe 註冊回調的地方：

useEffect(() => {
  if (checkIfSnapshotChanged(inst)) {
    forceUpdate({inst});
  }
  const handleStoreChange = () => {
    if (checkIfSnapshotChanged(inst)) {
      forceUpdate({inst});
    }
  };
  return subscribe(handleStoreChange);
}, [subscribe]);

## 檢查快照變化：
### 在訂閱之前，React 再次調用 checkIfSnapshotChanged(inst)，檢查是否有狀態變化（例如在 useLayoutEffect 和 useEffect 之間是否有其他更新）。如果有變化，則觸發 forceUpdate。

## 訂閱 store：
subscribe(handleStoreChange) 調用您提供的 subscribe 函數（例如 Redux 的 store.subscribe），並將 handleStoreChange 註冊為回調。
subscribe 返回一個清除函數（unsubscribe），用於在組件卸載或 subscribe` 依賴改變時清除訂閱。

## 當 store 變化時：
當外部 store（如 Redux store）發生變化時，handleStoreChange 會被調用。它會檢查快照是否變化（checkIfSnapshotChanged(inst)），如果變化，則調用 forceUpdate({inst})，觸發組件重新渲染。

## 觸發時機：
useEffect 在 DOM 更新和瀏覽器繪製後執行。訂閱是在這個階段建立的，意味著 handleStoreChange 只有在 store 變化時才會被觸發（例如 Redux dispatch 了一個 action）。

. 重新渲染
當 forceUpdate({inst}) 被調用時：

4. React 重新渲染組件。
在渲染期間，const value = getSnapshot() 會再次執行，獲取最新的快照。
組件使用這個快照渲染 UI，確保顯示的數據與 store 一致。
useLayoutEffect 和 useEffect 再次執行，更新 inst 和檢查快照變化，循環繼續。
```


精簡版本
``` javascript
useLayoutEffect(() => {
  /*
  在 useLayoutEffect 階段，更新快照值並檢查是否有變化。
    - 若快照不同，則觸發 forceUpdate 重新渲染，確保組件使用最新的快照。
  */
  inst.value = value;
  inst.getSnapshot = getSnapshot;
  if (checkIfSnapshotChanged(inst)) {
    forceUpdate({inst});
  }
}, [subscribe, value, getSnapshot]);

function checkIfSnapshotChanged(inst) {
  const latestGetSnapshot = inst.getSnapshot;
  const prevValue = inst.value;
  try {
    const nextValue = latestGetSnapshot();
    return !is(prevValue, nextValue);
  } catch (error) {
    return true;
  }
}

useEffect(() => {
  /*
  在 useEffect 階段：
  - 在瀏覽器繪製結束後，再檢查一次快照資料是否為最新的。
  - 並透過 subscribe 的 callback 以監聽後續快照變更。
  - 若有快照變更，則觸發 forceUpdate 重新渲染，確保組件使用最新的快照。
  */

  if (checkIfSnapshotChanged(inst)) {
    forceUpdate({inst});
  }

  const handleStoreChange = () => {
    if (checkIfSnapshotChanged(inst)) {
      forceUpdate({inst});
    }
  };
  return subscribe(handleStoreChange);
}, [subscribe]);
```

