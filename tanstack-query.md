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

## OnlineManager的關鍵原始碼內容是什麼?

```ts
// 繼承自 Subscribable 類，用於管理監聽器
export class OnlineManager extends Subscribable<Listener> {
  #online = true // 私有變量，追蹤在線狀態
  #cleanup?: () => void // 清理函數
  #setup: SetupFn // 設置函數
}

事件監聽設置:
constructor() {
  super()
  this.#setup = (onOnline) => {
    if (!isServer && window.addEventListener) {
      // 設置 online/offline 事件監聽
/*
當有監聽器時自動設置事件監聽
當沒有監聽器時自動清理資源
使用 #cleanup 函數進行資源清理
*/
      const onlineListener = () => onOnline(true)
      const offlineListener = () => onOnline(false)
      
      window.addEventListener('online', onlineListener, false)
      window.addEventListener('offline', offlineListener, false)

      // 返回清理函數
      return () => {
        window.removeEventListener('online', onlineListener)
        window.removeEventListener('offline', offlineListener)
      }
    }
    return
  }
}

訂閱管理:
// 當第一個監聽器被添加時觸發
protected onSubscribe(): void {
  if (!this.#cleanup) {
    this.setEventListener(this.#setup)
  }
}

核心功能方法:
// 設置新的事件監聽器
setEventListener(setup: SetupFn): void {
  this.#setup = setup
  this.#cleanup?.()
  this.#cleanup = setup(this.setOnline.bind(this))
}

// 更新在線狀態
setOnline(online: boolean): void {
  const changed = this.#online !== online
  if (changed) {
    this.#online = online
    this.listeners.forEach((listener) => {
      listener(online)
    })
  }
}

// 獲取當前在線狀態
isOnline(): boolean {
  return this.#online
}
```

## onlineManager 的 onSubscribe什麼時候會被觸發?

``` ts
會在 QueryClient 的 mount() 方法中被觸發
mount(): void {
  this.#mountCount++
  if (this.#mountCount !== 1) return

  this.#unsubscribeOnline = onlineManager.subscribe(async (online) => {
    if (online) {
     // 當網路恢復在線時會：
     // 恢復暫停的 mutations (resumePausedMutations)
      await this.resumePausedMutations()
//通知 queryCache 網路恢復 (this.#queryCache.onOnline())
      this.#queryCache.onOnline()
    }
  })
}

onOnline(): void {
  notifyManager.batch(() => {
    this.getAll().forEach((query) => {
      query.onOnline()  // 通知每個 query
    })
  })
}

每個 Query 實例收到通知後會：
onOnline(): void {
  const observer = this.observers.find((x) => x.shouldFetchOnReconnect())
  observer?.refetch({ cancelRefetch: false })  // 重新獲取數據
  this.#retryer?.continue()  // 繼續之前暫停的請求
}

這樣的設計形成了一個完整的網路狀態響應鏈：
onlineManager → QueryClient → QueryCache → Query instances，確保當網路恢復時，所有需要更新的查詢都能得到適當的處理。

```

## QueryCacheTanStack Query 中扮演著什麼角色?

``` ts
中央數據存儲
export class QueryCache extends Subscribable<QueryCacheListener> {
  #queries: QueryStore  // Map<string, Query> 存儲所有查詢
}

管理所有查詢的中央存儲
通過 queryHash 追蹤和存儲所有 Query 實例
查詢生命週期管理

build<TQueryFnData, TError, TData, TQueryKey extends QueryKey>(
  client: QueryClient,
  options: QueryOptions,
  state?: QueryState<TData, TError>,
): Query<TQueryFnData, TError, TData, TQueryKey> {
  // 檢查是否已存在相同的查詢
  let query = this.get(queryHash)
  // 確保相同的查詢不會重複創建
  if (!query) {
    query = new Query({...})
    this.add(query)
  }
  return query
}

統一管理和分發網路狀態變化
批量處理查詢更新

onOnline(): void {
  notifyManager.batch(() => {
    this.getAll().forEach((query) => {
      query.onOnline()
    })
  })
}
```

## 什麼時候onlineManager的setEventListener會被觸發

QueryClient.mount()
  → onlineManager.subscribe()
    → onlineManager.onSubscribe()
      → onlineManager.setEventListener(this.#setup)
        → 設置瀏覽器 online/offline 事件監聽

## onlineManager的狀態變更後，會做什麼?

```ts
constructor() {
  super()
  this.#setup = (onOnline) => {
    if (!isServer && window.addEventListener) {
      const onlineListener = () => onOnline(true)   // 這裡的 onOnline 就是 setOnline
      const offlineListener = () => onOnline(false)
      
      window.addEventListener('online', onlineListener, false)
      window.addEventListener('offline', offlineListener, false)
      // ...
    }
  }
}

setOnline(online: boolean): void {
  const changed = this.#online !== online
  if (changed) {
    this.#online = online
    this.listeners.forEach((listener) => {
      listener(online)
    })
  }
}
```
## 為什麼QueryClient 要檢查 mountCount > 1

這種設計的目的是：

防止重複訂閱：當同一個 QueryClient 在不同地方被 mount 時，只在第一次設置事件監聽
引用計數：使用 mountCount 追蹤 QueryClient 的使用狀態
資源管理：只有當所有 mount 都被 unmount 後（mountCount = 0），才真正清理資源
實際應用場景：

在 React 應用中，同一個 QueryClient 可能被多個組件共用
這些組件可能在不同時間 mount/unmount
mountCount 確保只有當最後一個使用者停止使用時，才清理資源


針對QueryClient的原始碼做解析，並用原始碼搭配註解回答

## mount/unmount是由誰觸發的?

由框架內部自動處理的，主要通過 QueryClientProvider 組件來實現。從代碼中可以看到：

``` ts
// packages/react-query/src/QueryClientProvider.tsx
React.useEffect(() => {
  client.mount()
  return () => {
    client.unmount()  // 在 cleanup function 中自動 unmount
  }
})

// packages/solid-query/src/QueryClientProvider.tsx
props.client.mount()
return props.client.unmount.bind(props.client)

當開發者使用 TanStack Query 時：

首先需要建立 QueryClient：
const queryClient = new QueryClient()

然後用 QueryClientProvider 包裹應用：
<QueryClientProvider client={queryClient}>
  <App />
</QueryClientProvider>

Provider 會自動處理：
在組件掛載時調用 client.mount()
在組件卸載時調用 client.unmount()
開發者不需要手動調用這些方法

多次 mount 的情況通常發生在：

同一個 QueryClient 被多個 Provider 使用
使用了動態導入的組件
在測試環境中重複創建 Provider
但這些都由框架內部處理，開發者不需要關心 mount/unmount 的具體邏輯。
```

## 如果開發者建立多個queryClient會發生什麼事?

```ts
緩存分散問題：

不推薦使用多個 QueryClient 實例主要有以下原因：
相同的查詢會在不同的 client 中重複存儲
無法共享查詢結果，導致重複的網路請求
<QueryClientProvider client={new QueryClient()}>
  <SomeComponent>
    {/* 這裡的查詢結果被存在第一個 QueryClient */}
    <Query key="users" />
  </SomeComponent>
  <QueryClientProvider client={new QueryClient()}>
    <OtherComponent>
      {/* 即使是相同的查詢，也會重新請求並存在第二個 QueryClient */}
      <Query key="users" />
    </OtherComponent>
  </QueryClientProvider>
</QueryClientProvider>

// Component A (使用第一個 QueryClient)
const { data: users } = useQuery('users', fetchUsers)
// 更新了用戶數據

// Component B (使用第二個 QueryClient)
const { data: users } = useQuery('users', fetchUsers)
// 仍然是舊的數據，因為緩存不共享

不同 QueryClient 之間的狀態無法同步
可能導致 UI 顯示不一致
資源效率問題：
每個 QueryClient 都維護自己的：
查詢緩存 (QueryCache)
變異緩存 (MutationCache)
事件監聽器
增加內存開銷
重複的網路請求增加頻寬使用
背景更新問題：
// 第一個 QueryClient 的查詢
useQuery('data', fetcher, {
  staleTime: 1000,
  refetchOnMount: true
})

// 第二個 QueryClient 的查詢
// 會觸發新的請求，因為它有自己的 staleTime 計算
useQuery('data', fetcher, {
  staleTime: 1000,
  refetchOnMount: true
})

每個 client 獨立管理重新獲取邏輯
可能導致過多的背景更新
建議做法是：

// ✅ 創建一個全局的 QueryClient 實例
const queryClient = new QueryClient()

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <SomeComponent />
      <OtherComponent />
    </QueryClientProvider>
  )
}

typescript


這樣可以：

共享查詢緩存
確保數據一致性
優化資源使用
統一管理背景更新策略

```


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


如果沒有 QueryCache：

數據重複：
相同的查詢可能會重複創建多個實例
造成不必要的網絡請求和內存消耗
狀態不一致：
無法在不同組件間共享查詢狀態
可能導致 UI 顯示不一致的數據
事件處理混亂：
網路狀態變化需要單獨通知每個查詢
無法進行批量優化處理
性能問題：
失去查詢復用機制
每次都需要重新發起請求
QueryCache 本質上是一個中央協調器，確保查詢的一致性、效率和可靠性。
