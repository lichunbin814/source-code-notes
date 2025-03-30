# Jotai

### useAtom的核心實現邏輯是什麼?

#### 關鍵點總結
 - useAtom 組合了 useAtomValue 和 useSetAtom 兩個 hooks
 - useAtomValue 負責讀取數據並返回當前值
 - useSetAtom 提供更新 atom 值的函數
 - 使用 store 來統一管理 atom 的狀態
 - useReducer 負責觸發 UI 的重新渲染
 - useEffect 用於訂閱 atom 的變化
 - WeakMap 用於管理異步 Promise 狀態
 - 支持同步和異步的 atom 數值類型
 - 提供可選的延遲更新功能
 - 使用 useCallback 優化 setter 的效能
 - atom 的訂閱機制確保數據同步更新

```ts
// 核心實現邏輯整合
/* 
主要的核心邏輯分成三個部分:

1. useAtom Hook
- 結合 useAtomValue 和 useSetAtom
- 返回一個元組 [value, setter]
*/
export function useAtom<Value, Args extends unknown[], Result>(
  atom: Atom<Value> | WritableAtom<Value, Args, Result>,
  options?: Options,
) {
  return [
    useAtomValue(atom, options),
    useSetAtom(atom as WritableAtom<Value, Args, Result>, options),
  ]
}

/* 
2. useAtomValue Hook 的關鍵邏輯
- 使用 useReducer 管理狀態更新
- 處理 async atoms (Promise)
- 支持延遲更新(delay option)
- 使用 store.sub 訂閱 atom 的變化
*/
function useAtomValue(atom, options) {
  const store = useStore(options)
  // useReducer 用於觸發重新渲染
  const [[value, storeFromReducer, atomFromReducer], rerender] = useReducer(/*...*/)
  
  // 訂閱 atom 變化
  useEffect(() => {
    const unsub = store.sub(atom, () => {
      rerender()
    })
    rerender()
    return unsub
  }, [store, atom])
  
  // 處理異步值
  if (isPromiseLike(value)) {
    const promise = createContinuablePromise(value, () => store.get(atom))
    return use(promise)
  }
  return value
}

/* 
3. useSetAtom Hook 的關鍵邏輯
- 返回一個 memoized 的 setter 函數
- 通過 store.set 更新 atom 值
- 有開發環境的型別檢查
*/
function useSetAtom(atom, options) {
  const store = useStore(options)
  const setAtom = useCallback(
    (...args) => {
      // 開發環境檢查 atom 是否可寫
      if (import.meta.env?.MODE !== 'production' && !('write' in atom)) {
        throw new Error('not writable atom')
      }
      return store.set(atom, ...args)
    },
    [store, atom],
  )
  return setAtom
}
```
