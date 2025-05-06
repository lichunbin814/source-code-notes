# Tanstack Query

## 基本工作流程

應用初始化時創建 QueryClient 實例
QueryClientProvider 將該實例注入 React Context
useQuery Hook:
通過 Context 獲取 QueryClient
創建 QueryObserver 實例監聽查詢
QueryObserver 通過 QueryClient 執行查詢
將查詢結果返回給組件

## QueryClientProvider是什麼

``` ts
是一個 React Context Provider
接收 QueryClient 實例作為 prop
使子組件可以訪問到 QueryClient 實例
架構如下:
<QueryClientProvider client={queryClient}>
  {children} // 可以使用 useQuery 的組件
</QueryClientProvider>
```

### 原始碼分析

``` ts
'use client'  // React Server Components 標記

import * as React from 'react'
import type { QueryClient } from '@tanstack/query-core'

// 1. 創建 Context
export const QueryClientContext = React.createContext<QueryClient | undefined>(
  undefined,
)

// 2. Hook: 用於獲取 QueryClient 實例
export const useQueryClient = (queryClient?: QueryClient) => {
  const client = React.useContext(QueryClientContext)

  if (queryClient) {
    return queryClient
  }

  if (!client) {
    throw new Error('No QueryClient set, use QueryClientProvider to set one')
  }

  return client
}

// 3. Provider Props 類型定義
export type QueryClientProviderProps = {
  client: QueryClient
  children?: React.ReactNode
}

// 4. Provider 組件實現
export const QueryClientProvider = ({
  client,
  children,
}: QueryClientProviderProps): React.JSX.Element => {
  // 5. 生命週期管理
  React.useEffect(() => {
    client.mount()
    return () => {
      client.unmount()
    }
  }, [client])

  // 6. 渲染 Provider
  return (
    <QueryClientContext.Provider value={client}>
      {children}
    </QueryClientContext.Provider>
  )
}

```

## useQueryClient是什麼?

```ts
export const useQueryClient = (queryClient?: QueryClient) => {
  const client = React.useContext(QueryClientContext)
  
  if (queryClient) {
    return queryClient
  }
  
  if (!client) {
    throw new Error('No QueryClient set, use QueryClientProvider to set one')
  }
  
  return client
}

```

## useQuery Hook 運作流程

``` ts
const { isPending, error, data } = useQuery({
  queryKey: ['repoData'],
  queryFn: () => fetch('...').then(res => res.json())
})

主要數據流:

組件使用 useQuery 發起查詢
QueryObserver 通過 QueryClient 檢查 cache
如果需要則執行 queryFn 獲取新數據
將數據存入 QueryCache
QueryObserver 監聽到更新並通知組件
```

## QueryObserver在做什麼?
``` ts
觀察者模式,監聽查詢狀態變化
class QueryObserver extends Subscribable {
  #client: QueryClient
  #currentQuery: Query
  #currentResult: QueryObserverResult
  
  constructor(client: QueryClient, options: QueryObserverOptions) {
    this.#client = client
    this.setOptions(options) // 設置查詢選項
  }
}
```

##  QueryClient 的 mount 和 unmount 在做什麼?
``` ts
/**
 * mount 方法：處理 QueryClient 的掛載邏輯
 * 主要功能：
 * 1. 追蹤掛載次數
 * 2. 註冊視窗焦點和網路狀態的監聽器
 * 3. 管理查詢和突變的生命週期
 */
mount(): void {
    // 增加掛載計數器
    this.#mountCount++
    
    // 如果不是第一次掛載，直接返回
    // 這樣可以確保事件監聽器只註冊一次，避免重複註冊
    if (this.#mountCount !== 1) return

    // 註冊視窗焦點變化監聽器
    this.#unsubscribeFocus = focusManager.subscribe(async (focused) => {
      if (focused) {  // 當視窗重新獲得焦點時
        // 1. 首先恢復所有暫停的突變操作
        await this.resumePausedMutations()
        // 2. 觸發查詢緩存的焦點事件，可能會重新驗證過期的查詢
        this.#queryCache.onFocus()
      }
    })

    // 註冊網路狀態變化監聽器
    this.#unsubscribeOnline = onlineManager.subscribe(async (online) => {
      if (online) {  // 當網路重新連線時
        // 1. 恢復之前因離線而暫停的突變操作
        await this.resumePausedMutations()
        // 2. 觸發查詢緩存的上線事件，可能會重新執行失敗的查詢
        this.#queryCache.onOnline()
      }
    })
}

/**
 * unmount 方法：處理 QueryClient 的卸載邏輯
 * 主要功能：
 * 1. 追蹤卸載次數
 * 2. 清理事件監聽器
 * 3. 釋放資源
 */
unmount(): void {
    // 減少掛載計數器
    this.#mountCount--
    
    // 如果還有其他實例在運行，直接返回
    // 這確保只有在最後一個實例卸載時才清理資源
    if (this.#mountCount !== 0) return

    // 清理視窗焦點監聽器
    // 使用可選鏈運算符 ?. 安全地呼叫取消訂閱函數
    this.#unsubscribeFocus?.()
    this.#unsubscribeFocus = undefined

    // 清理網路狀態監聽器
    this.#unsubscribeOnline?.()
    this.#unsubscribeOnline = undefined
}
```

### QueryClinet的單例模式設計解析

讓 React Query 能夠在保持高效能的同時，還確保了數據的一致性和資源的合理使用。


``` ts
優點
資源共享與效率

所有組件共用同一個查詢緩存
避免重複的數據請求
減少不必要的 API 調用
狀態一致性

確保所有組件看到相同的數據
避免數據不同步問題
集中管理數據更新
事件監聽優化

全局事件只註冊一次
避免重複的事件監聽器
降低記憶體使用

class QueryClient {
  #mountCount = 0  // 追蹤實例掛載次數
  #queryCache: QueryCache  // 集中管理查詢緩存
  #mutationCache: MutationCache  // 集中管理突變緩存
  
  mount(): void {
    this.#mountCount++
    if (this.#mountCount !== 1) return
    // 只在第一次掛載時註冊全局事件
  }
  
  unmount(): void {
    this.#mountCount--
    if (this.#mountCount !== 0) return
    // 只在最後一次卸載時清理資源
  }
}
```

問題防範

``` ts
防止記憶體洩漏

unmount(): void {
  this.#mountCount--
  if (this.#mountCount === 0) {
    // 確保資源完全清理
    this.#unsubscribeFocus?.()
    this.#unsubscribeOnline?.()
  }
}

mount(): void {
  this.#mountCount++
  if (this.#mountCount !== 1) return
  // 防止多次註冊事件監聽
}
```
### 如果UseClient沒有控制「重複註冊事件監聽器的問題與影響」，會怎麼樣?


```ts
1. 效能問題
// 錯誤示例：未控制重複註冊
class BadQueryClient {
  mount() {
    focusManager.subscribe((focused) => {
      if (focused) {
        this.queryCache.onFocus()  // 會重複執行
      }
    })
  }
}

// 當視窗獲得焦點時：
window.focus() 觸發 -> {
  queryCache.onFocus()  // 第1次執行
  queryCache.onFocus()  // 第2次執行
  queryCache.onFocus()  // 第3次執行...
}

2. 記憶體洩漏
// 每次 mount 都新增監聽器，但沒有相應的清理機制
mount() {
  this.#mountCount++  // 計數增加
  
  // 監聽器不斷累積
  focusManager.subscribe(...)  // 監聽器 1
  focusManager.subscribe(...)  // 監聽器 2
  focusManager.subscribe(...)  // 監聽器 3
}

造成：

監聽器持續累積在記憶體中
無法被垃圾回收
記憶體使用量持續增長

3. React Query 的解決方案
class QueryClient {
  #mountCount = 0
  #unsubscribeFocus?: () => void
  
  mount(): void {
    this.#mountCount++
    if (this.#mountCount !== 1) return  // 關鍵：只在第一次註冊
    
    this.#unsubscribeFocus = focusManager.subscribe(...)  // 保存清理函數
  }
  
  unmount(): void {
    this.#mountCount--
    if (this.#mountCount === 0) {  // 關鍵：只在最後一次清理
      this.#unsubscribeFocus?.()
      this.#unsubscribeFocus = undefined
    }
  }
}
```

## 使用者在切換頁面時，QueryClient會做什麼事?


```ts
當使用者切換頁面回來時，QueryClient 會執行以下操作：
mount(): void {
  this.#mountCount++
  if (this.#mountCount !== 1) return

  this.#unsubscribeFocus = focusManager.subscribe(async (focused) => {
    if (focused) {
      // 1. 恢復被暫停的 mutations
      await this.resumePausedMutations()
      // 2. 觸發 queryCache 的 onFocus 事件
      this.#queryCache.onFocus()
    }
  })
}

恢復暫停的 Mutations：
resumePausedMutations(): Promise<unknown> {
  if (onlineManager.isOnline()) {
    return this.#mutationCache.resumePausedMutations()
  }
  return Promise.resolve()
}

resumePausedMutations(): Promise<unknown> {
  // 1. 找出所有被暫停的 mutations
  const pausedMutations = this.getAll().filter((x) => x.state.isPaused)

  // 2. 使用 notifyManager.batch 批次處理
  return notifyManager.batch(() =>
    Promise.all(
      // 3. 對每個暫停的 mutation 調用 continue()
      pausedMutations.map((mutation) => 
        mutation.continue().catch(noop)
      ),
    )
  )
}
```

## FoucsManager在做什麼?
focusManager 通過以下機制監聽瀏覽器 visibility 狀態(tab是否active):
``` ts
constructor() {
  this.#setup = (onFocus) => {
    if (!isServer && window.addEventListener) {
      // 註冊 visibilitychange 事件監聽
      window.addEventListener('visibilitychange', listener, false)
      
      return () => {
        // 清理事件監聽
        window.removeEventListener('visibilitychange', listener)
      }
    }
    return
  }

isFocused(): boolean {
  if (typeof this.#focused === 'boolean') {
    return this.#focused
  }
  // 使用 document.visibilityState 判斷頁面是否可見
  return globalThis.document?.visibilityState !== 'hidden'
}

onFocus(): void {
  const isFocused = this.isFocused()
  // 通知所有訂閱者頁面焦點狀態變更
  this.listeners.forEach((listener) => {
    listener(isFocused)
  })
}
```

## 在什麼時候focusManager會把state改為paused?
```ts


相同 scope 的其他 mutation 正在執行時
// mutationCache.ts
canRun(mutation: Mutation<any, any, any, any>): boolean {
  const scope = scopeFor(mutation)
  if (typeof scope === 'string') {
    const mutationsWithSameScope = this.#scopes.get(scope)
    const firstPendingMutation = mutationsWithSameScope?.find(
      (m) => m.state.status === 'pending'  // 尋找相同 scope 中是否有正在執行的 mutation
    )
    // 只有當沒有其他 pending mutation 或自己是第一個 pending mutation 時才能執行
    return !firstPendingMutation || firstPendingMutation === mutation
  }
  return true
}


retryer 判斷無法開始執行時
// 1. canStart 用於初始檢查是否可以開始執行
const canStart = () => canFetch(config.networkMode) && config.canRun()

// 2. canContinue 用於檢查是否可以繼續執行
const canContinue = () =>
  focusManager.isFocused() &&
  (config.networkMode === 'always' || onlineManager.isOnline()) &&
  config.canRun()

// 3. 關鍵的暫停邏輯在 run 方法中
Promise.resolve(promiseOrValue)
  .catch((error) => {
    // ...retry 邏輯...
    
    sleep(delay)
      .then(() => {
        // 這裡是關鍵：檢查是否可以繼續，如果不行就調用 pause()
        return canContinue() ? undefined : pause()
      })
      .then(() => {
        if (isRetryCancelled) {
          reject(error)
        } else {
          run()
        }
      })
  })

// 4. pause 方法實際觸發狀態變更
const pause = () => {
  return new Promise((continueResolve) => {
    continueFn = (value) => {
      if (isResolved || canContinue()) {
        continueResolve(value)
      }
    }
    // 這裡觸發 onPause callback，最終導致 mutation 狀態變為 paused
    config.onPause?.()
  }).then(() => {
    continueFn = undefined
    if (!isResolved) {
      config.onContinue?.()
    }
  })
}

function canFetch(networkMode: NetworkMode | undefined): boolean {
//網路離線時（通過 networkMode 配置）
  return (networkMode ?? 'online') === 'online'
    ? onlineManager.isOnline()  // 檢查是否在線上
    : true
}

```

## scope和mutation分別在做什麼些什麼?

```ts
Mutation 是指資料的修改操作，例如：
Scope 則是用來管理相關的 mutations 執行順序

// 使用者更新資料的 mutation
// useMutation類似於queue
const updateUserMutation = useMutation({
  mutationFn: updateUser,
  // 設定 scope，確保同一個使用者的更新操作不會同時執行
  scope: { id: 'user', userId: '123' }
})

// 實際使用案例
const handleSubmit = () => {
  // 同時觸發多次更新
  // 當第一個 mutation 在執行時，其他相同 scope 的 mutations 會進入暫停狀態
等第一個完成後，第二個才會開始執行，以此類推
mutation.mutate({ title: 'New Title' })     // 第一個執行
mutation.mutate({ content: 'New Content' }) // 進入 queue
mutation.mutate({ tags: ['new', 'tag'] })  // 進入 queue
}

這樣的設計有幾個好處：
避免同時對同一資源進行多次修改造成資料不一致
確保更新操作的順序性
防止不必要的競爭條件

類似 queue 的行為，實現邏輯：

// mutationCache.ts
canRun(mutation: Mutation<any, any, any, any>): boolean {
  const scope = scopeFor(mutation)
  if (typeof scope === 'string') {
    const mutationsWithSameScope = this.#scopes.get(scope)
    const firstPendingMutation = mutationsWithSameScope?.find(
      (m) => m.state.status === 'pending'
    )
    // 只有 queue 中第一個可以執行
 // 兩種情況可以執行：
    // 1. 沒有其他 pending mutation (!firstPendingMutation)
    // 2. 這個 mutation 就是第一個 pending mutation (firstPendingMutation === mutation)
    return !firstPendingMutation || firstPendingMutation === mutation
  }
  return true
}

// 當一個 mutation 完成時，會嘗試執行 queue 中的下一個
runNext(mutation: Mutation<any, any, any, any>): Promise<unknown> {
  const scope = scopeFor(mutation)
  if (typeof scope === 'string') {
    // 尋找 queue 中下一個被暫停的 mutation
    const foundMutation = this.#scopes
      .get(scope)
      ?.find((m) => m !== mutation && m.state.isPaused)

    // 繼續執行找到的下一個 mutation
    return foundMutation?.continue() ?? Promise.resolve()
  }
  return Promise.resolve()
}

```


針對QueryClient的原始碼做解析，並回答



解釋resumePausedMutations

為什麼要觸發查詢緩存的焦點事件，可能會重新驗證過期的查詢

onlineManager、#queryCache.onOnline、resumePausedMutations怎麼實現的

onFocus的實現邏輯是什麼

mountCount大於1是什麼情境

unsubscribeFocus、unsubscribeOnline怎麼實現的

分析retryer.js是怎麼實現的

分析 mutationCache 是怎麼實現的
