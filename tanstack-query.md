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
網路離線時（通過 networkMode 配置）
相同 scope 的其他 mutation 正在執行時
retryer 判斷無法開始執行時
```


針對QueryClient的原始碼做解析，並回答


解釋resumePausedMutations

為什麼要觸發查詢緩存的焦點事件，可能會重新驗證過期的查詢

onlineManager、#queryCache.onOnline、resumePausedMutations怎麼實現的

onFocus的實現邏輯是什麼

mountCount大於1是什麼情境

unsubscribeFocus、unsubscribeOnline怎麼實現的
