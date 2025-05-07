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

## retryer 裡的pause 方法裡的config.onPause 怎麼對應到mutation的

Mutation 創建時會有唯一的 mutationId
每個 Mutation 實例有自己的 retryer 實例
Retryer 建立時，會得到該 mutation 提供的 config（包含 onPause callback）
當需要暫停時，retryer 呼叫 config.onPause，就會觸發對應 mutation 的 dispatch({ type: 'pause' })

``` ts
retryer 中的 config.onPause 是透過 mutation 的 execute 方法設置的。讓我們看看整個流程：

// mutation.ts
async execute(variables: TVariables): Promise<TData> {
  // 1. 建立 retryer，並傳入 config
  this.#retryer = createRetryer({
    fn: () => {
      if (!this.options.mutationFn) {
        return Promise.reject(new Error('No mutationFn found'))
      }
      return this.options.mutationFn(variables)
    },
    onPause: () => {
      // 2. 設定 onPause callback
        // 這裡的 this 指向建立 retryer 的那個 mutation 實例

      this.#dispatch({ type: 'pause' })
    },
    // ... 其他配置
  })

  // 3. 檢查是否需要暫停
  const isPaused = !this.#retryer.canStart()
}

當 retryer 呼叫 pause() 時：

// retryer.ts
const pause = () => {
  return new Promise((continueResolve) => {
    continueFn = (value) => {
      if (isResolved || canContinue()) {
        continueResolve(value)
      }
    }
    // 4. 呼叫在 mutation execute 時設定的 onPause callback
    config.onPause?.()
  })
}
```

## mutaion扮演了什麼角色，沒有他會發生什麼事?
``` ts
資料變更管理
Mutation 負責處理所有的資料變更操作(Create, Update, Delete)
提供完整的狀態管理，包括 pending, success, error 等狀態
追蹤重要的元數據，如 failureCount, isPaused, submittedAt 等

錯誤處理與重試機制

內建重試機制
提供錯誤追蹤和恢復機制
可配置的重試次數和延遲

retry: this.options.retry ?? 0,
retryDelay: this.options.retryDelay

生命週期管理
await this.options.onMutate?.(variables)
await this.options.onSuccess?.(data, variables, this.state.context!)
await this.options.onError?.(error as TError, variables, this.state.context)
await this.options.onSettled?.(data, null, variables, this.state.context)
```
如果沒有 Mutation 會發生什麼?
``` ts
缺乏狀態管理
開發者需要手動追蹤請求狀態
沒有統一的載入、成功、錯誤狀態管理
需要自己實現樂觀更新邏輯

```

## 如何確保從別的tab切換回來，頁面依然保持最新資料

``` ts
透過focusManager
this.#unsubscribeFocus = focusManager.subscribe(async (focused) => {
  if (focused) {
    await this.resumePausedMutations()
    this.#queryCache.onFocus()
  }
})

onFocus(): void {
  const isFocused = this.isFocused()
  // 通知所有訂閱者頁面焦點狀態變更
  this.listeners.forEach((listener) => {
    listener(isFocused)
  })
}
```

## 當建立多個 QueryClient 時，focusManager 的 listener會是多個嗎?

但官方強烈建議使用單一 QueryClient!! 以下只是探討邊界情況

``` ts
每個 QueryClient 實例都有自己獨立的 mountCount：

export class QueryClient {
  #mountCount: number

  constructor(config: QueryClientConfig = {}) {
    this.#mountCount = 0
  }
}

當一個 QueryClient 實例呼叫 mount() 時：
mount(): void {
  this.#mountCount++
  // 這個 mountCount 檢查只確保「同一個」QueryClient 實例不會重複註冊 listener。
  if (this.#mountCount !== 1) return

  this.#unsubscribeFocus = focusManager.subscribe(async (focused) => {
    if (focused) {
      await this.resumePausedMutations()
      this.#queryCache.onFocus()
    }
  })
}

每個 QueryClient 實例都會在自己首次 mount 時註冊一個新的 listener 到 focusManager 的 listeners Set 中：
// 在 subscribable.ts
protected listeners = new Set<TListener>()


因此，如果同時存在多個 QueryClient 實例，focusManager 就會有多個 listener。每個 QueryClient 實例會貢獻一個 listener，而且這些 listener 都是獨立的函數，所以會被分別存儲在 Set 中。
```

## QueryClient的角色是什麼? 為什麼要有他?

``` ts
用兩個 API 的例子來說明 QueryClient 的角色：

// 定義兩個 API
const fetchTest1 = async () => {
  const response = await fetch('/api/test1')
  return response.json()
}

const fetchTest2 = async () => {
  const response = await fetch('/api/test2')
  return response.json()
}

// 建立一個 QueryClient 來管理所有 API
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30000,  // 所有 API 共用的預設設定
    }
  }
})

// 在不同元件中使用這些 API
function Component1() {
  // 使用 test1 API
  const { data: test1Data } = useQuery(['test1'], fetchTest1)
  return <div>{test1Data}</div>
}

function Component2() {
  // 使用 test2 API
  const { data: test2Data } = useQuery(['test2'], fetchTest2)
  return <div>{test2Data}</div>
}


QueryClient 對這些 API 的管理角色：

統一的快取管理：
// 可以手動操作任何 API 的快取
queryClient.setQueryData(['test1'], newData)
queryClient.invalidateQueries(['test2'])

// 也可以一次操作多個相關的 API
queryClient.invalidateQueries(['test']) // 使所有以 test 開頭的查詢失效

typescript

⟼

共用的狀態監控：
// 監控所有 API 的狀態
const isFetching = queryClient.isFetching() // 是否有任何 API 正在取得資料

typescript

⟼

相依性管理：
// test2 依賴 test1 的資料
const { data: test2Data } = useQuery(
  ['test2'],
  fetchTest2,
  {
    enabled: !!queryClient.getQueryData(['test1']) // 只有當 test1 有資料時才執行
  }
)

typescript

⟼

一致的錯誤處理：
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 3,        // 所有 API 失敗時都重試 3 次
      retryDelay: 1000 // 重試間隔 1 秒
    }
  }
})

typescript

⟼

全域的資料更新：
// 當用戶切換回視窗時，可以選擇更新某些 API
queryClient.mount() // 這會透過 focusManager 監聽焦點
// 當視窗重新獲得焦點時
queryClient.invalidateQueries(['test1']) // 更新 test1
queryClient.invalidateQueries(['test2']) // 更新 test2

typescript

⟼

所以不管有多少個 API：

它們共享同一個 QueryClient 實例
統一的設定和管理介面
一致的快取和重試策略
集中的狀態管理
統一的事件響應（如視窗焦點變化）
這就是為什麼通常只需要一個 QueryClient - 它就像是所有 API 的管理中心。

```

針對QueryClient的原始碼做解析，並回答


onlineManager、#queryCache.onOnline、resumePausedMutations怎麼實現的

onFocus的實現邏輯是什麼

mountCount大於1是什麼情境

unsubscribeFocus、unsubscribeOnline怎麼實現的

分析retryer.js是怎麼實現的

分析 mutationCache 是怎麼實現的

深入解析mutaion的這些機制是怎麼做的?
變更操作(Create, Update, Delete)

狀態管理包括 pending, success, error 等狀態
追蹤重要的元數據，如 failureCount, isPaused, submittedAt 等
生命週期管理
樂觀更新邏輯
去重和防抖機制
競態條件(race condition)
並發控制機制
追蹤請求狀態
標準化的錯誤處理
導致不必要的重新渲染
自動的資源清理機制
缺乏統一的資料變更模式
