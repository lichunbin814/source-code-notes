# Tanstack Query

## 用最簡單的方式去實作核心邏輯

```ts
// 使用示例
const todoQuery = new SimpleQuery()

// 執行查詢
todoQuery.fetch(() => 
  fetch('/todos').then(r => r.json())
).then(state => {
  if (state.isLoading) {
    console.log('Loading...')
  } else if (state.error) {
    console.log('Error:', state.error)
  } else {
    console.log('Data:', state.data)
  }
})

// 最簡單的查詢類
class SimpleQuery {
  constructor() {
    this.state = {
      data: null,
      error: null,
      isLoading: false
    }
  }

  // 執行查詢
  async fetch(queryFn) {
    try {
      // 1. 開始加載
      this.state.isLoading = true
      
      // 2. 執行查詢
      const data = await queryFn()
      
      // 3. 更新狀態
      this.state = {
        data,
        error: null,
        isLoading: false
      }
      
      return this.state
      
    } catch (error) {
      // 4. 處理錯誤
      this.state = {
        data: null,
        error,
        isLoading: false
      }
      
      return this.state
    }
  }
}


```

加入緩存功能
``` ts


// 使用示例
const queryCache = new SimpleQueryCache()

// 執行查詢
async function fetchQuery(queryKey, queryFn) {
  // 1. 獲取或創建查詢
  const query = queryCache.build(queryKey, queryFn)
  
  try {
    // 2. 執行查詢函數
    const data = await query.queryFn()
    
    // 3. 更新查詢狀態
    query.state = {
      data,
      error: null,
      status: 'success'
    }
    
    return query.state
    
  } catch (error) {
    // 4. 處理錯誤
    query.state = {
      data: undefined,
      error,
      status: 'error'
    }
    
    throw error
  }
}

// 使用示例
const query = fetchQuery('todos', () => 
  fetch('/todos').then(r => r.json())
)

// 簡化版的 QueryCache
class SimpleQueryCache {
  constructor() {
    this.queries = new Map()  // 使用 Map 存儲查詢，與原碼一致
  }

  // 類似原始碼的 build 方法
  build(queryKey, queryFn) {
    // 生成 queryHash（簡化版直接使用 key）
    const queryHash = queryKey
    
    // 檢查是否存在查詢
    let query = this.get(queryHash)
    
    // 如果不存在，創建新的查詢
    if (!query) {
      query = {
        queryHash,
        queryKey,
        queryFn,
        state: {
          data: undefined,
          error: null,
          status: 'pending'
        }
      }
      this.add(query)
    }
    
    return query
  }

  // 對應原始碼的 add 方法
  add(query) {
    if (!this.queries.has(query.queryHash)) {
      this.queries.set(query.queryHash, query)
    }
  }

  // 對應原始碼的 get 方法
  get(queryHash) {
    return this.queries.get(queryHash)
  }

  // 對應原始碼的 remove 方法
  remove(queryHash) {
    this.queries.delete(queryHash)
  }
}
```

處理緩存過期(staleTime)
```ts
class SimpleQueryWithStale {
  constructor() {
    this.queries = new Map()
  }

  // 簡化版的 timeUntilStale 函數
  timeUntilStale(updatedAt, staleTime) {
    return Math.max(updatedAt + (staleTime || 0) - Date.now(), 0)
  }

  // 檢查數據是否過期
  isStale(query) {
    return this.timeUntilStale(query.state.dataUpdatedAt, query.staleTime) === 0
  }

  async fetch(queryKey, queryFn, options = {}) {
    const existingQuery = this.queries.get(queryKey)
    
    // 檢查緩存是否可用（未過期）
    if (existingQuery && !this.isStale(existingQuery)) {
      return existingQuery.state
    }

    // 創建或更新查詢
    const query = {
      queryKey,
      queryFn,
      staleTime: options.staleTime || 0,  // 過期時間，默認為 0
      state: {
        data: null,
        error: null,
        isLoading: true,
        dataUpdatedAt: 0     // 數據更新時間戳
      }
    }

    this.queries.set(queryKey, query)

    try {
      // 執行查詢
      const data = await queryFn()
      
      // 更新查詢狀態
      query.state = {
        data,
        error: null,
        isLoading: false,
        dataUpdatedAt: Date.now()  // 設置數據更新時間
      }
      
      return query.state
      
    } catch (error) {
      query.state = {
        data: null,
        error,
        isLoading: false,
        dataUpdatedAt: 0
      }
      throw error
    }
  }
}

// 使用示例
const queryClient = new SimpleQueryWithStale()

// 使用方式
async function fetchTodos() {
  // 首次獲取，設置 5 秒過期時間
  const result1 = await queryClient.fetch('todos', 
    () => fetch('/todos').then(r => r.json()),
    { staleTime: 5000 }
  )
  console.log('First fetch:', result1)

  // 1 秒後再次獲取（使用緩存）
  setTimeout(async () => {
    const result2 = await queryClient.fetch('todos', 
      () => fetch('/todos').then(r => r.json()),
      { staleTime: 5000 }
    )
    console.log('Second fetch (from cache):', result2)
  }, 1000)

  // 6 秒後再次獲取（緩存過期，重新請求）
  setTimeout(async () => {
    const result3 = await queryClient.fetch('todos', 
      () => fetch('/todos').then(r => r.json()),
      { staleTime: 5000 }
    )
    console.log('Third fetch (stale, new request):', result3)
  }, 6000)
}

```

數據轉換
```
class SimpleQueryWithSelect {
  constructor() {
    this.queries = new Map()
  }

  async fetch(queryKey, queryFn, options = {}) {
    // 1. 創建或獲取查詢
    let query = {
      queryKey,
      queryFn,
      state: {
        data: null,
        error: null,
        isLoading: true,
        dataUpdatedAt: 0
      }
    }

    try {
      // 2. 執行查詢
      const rawData = await queryFn()
      
      // 3. 更新原始數據
      query.state = {
        data: rawData,
        error: null,
        isLoading: false,
        dataUpdatedAt: Date.now()
      }

      // 4. 應用數據轉換（select）
      const transformedData = options.select 
        ? options.select(rawData)
        : rawData

      // 5. 返回結果
      return {
        data: transformedData,
        error: null,
        isLoading: false,
        // 保留原始數據
        rawData: rawData
      }

    } catch (error) {
      query.state = {
        data: null,
        error,
        isLoading: false,
        dataUpdatedAt: 0
      }
      throw error
    }
  }
}

// 使用示例：
const queryClient = new SimpleQueryWithSelect()

// 基本查詢
const basicResult = await queryClient.fetch(
  'todos',
  () => fetch('/todos').then(r => r.json())
)

// 帶數據轉換的查詢
const transformedResult = await queryClient.fetch(
  'todos',
  () => fetch('/todos').then(r => r.json()),
  {
    // 轉換函數
    select: (data) => ({
      items: data.map(todo => ({
        ...todo,
        title: todo.title.toUpperCase()  // 轉換 title 為大寫
      })),
      total: data.length
    })
  }
)

console.log(transformedResult)
// {
//   data: {
//     items: [{ title: 'TODO 1' }, { title: 'TODO 2' }],
//     total: 2
//   },
//   rawData: [{ title: 'todo 1' }, { title: 'todo 2' }],
//   error: null,
//   isLoading: false
// }

```

錯誤處理
```ts
// 最簡單的查詢類
class SimpleQuery {
  constructor() {
    this.state = {
      data: null,
      error: null,      // 存儲錯誤信息
      isLoading: false
    }
  }

  // 執行查詢
  async fetch(queryFn, options = {}) {
    // 開始加載
    this.state.isLoading = true
    
    try {
      // 嘗試執行查詢
      const data = await queryFn()
      
      // 成功：更新狀態
      this.state = {
        data,
        error: null,
        isLoading: false
      }

      // 可選：調用成功回調
      options.onSuccess?.(data)
      
      return this.state
      
    } catch (error) {
      // 構建錯誤狀態
      const errorState = {
        data: null,
        error,
        isLoading: false
      }

      // 檢查是否需要拋出錯誤
      if (options.throwOnError) {
        throw error
      }

      // 更新狀態
      this.state = errorState
      
      // 可選：調用錯誤回調
      options.onError?.(error)
      
      return errorState
    }
  }
}

// 使用示例
const query = new SimpleQuery()

// 基本使用
try {
  const result = await query.fetch(
    () => fetch('/api/data'),
    {
      // 錯誤處理選項
      throwOnError: true,     // 是否拋出錯誤
      onError: (error) => console.error('Query failed:', error),
      onSuccess: (data) => console.log('Query succeeded:', data)
    }
  )
} catch (error) {
  console.log('Caught error:', error)
}

```

重試機制
```ts
// 簡化版的重試邏輯
class SimpleQueryWithRetry {
  constructor() {
    this.state = {
      data: null,
      error: null,
      isLoading: false,
      failureCount: 0    // 追蹤失敗次數
    }
  }

  // 預設重試延遲計算（與原始碼相同）
  defaultRetryDelay(failureCount) {
    return Math.min(1000 * 2 ** failureCount, 30000)
  }

  async fetch(queryFn, options = {}) {
    this.state.isLoading = true
    this.state.failureCount = 0
    
    // 獲取重試配置（與原始碼邏輯類似）
    const retry = options.retry ?? 3
    const retryDelay = options.retryDelay ?? this.defaultRetryDelay

    const run = async () => {
      try {
        const data = await queryFn()
        
        this.state = {
          data,
          error: null,
          isLoading: false,
          failureCount: 0
        }
        
        return this.state
        
      } catch (error) {
        this.state.failureCount++

        // 檢查是否應該重試（簡化版的重試判斷邏輯）
        const shouldRetry = 
          retry === true || 
          (typeof retry === 'number' && this.state.failureCount < retry)

        if (!shouldRetry) {
          this.state = {
            data: null,
            error,
            isLoading: false,
            failureCount: this.state.failureCount
          }
          throw error
        }

        // 計算延遲時間
        const delay = typeof retryDelay === 'function'
          ? retryDelay(this.state.failureCount, error)
          : retryDelay(this.state.failureCount)

        // 等待後重試
        await new Promise(resolve => setTimeout(resolve, delay))
        return run()  // 遞迴重試
      }
    }

    return run()
  }
}

// 使用示例
const query = new SimpleQueryWithRetry()

// 基本使用
try {
  const result = await query.fetch(
    async () => {
      // 模擬可能失敗的請求
      const response = await fetch('/api/data')
      if (!response.ok) throw new Error('API Error')
      return response.json()
    },
    {
      retry: 3,                          // 最多重試 3 次
      retryDelay: (failureCount) => {    // 指數退避延遲
        return Math.min(1000 * 2 ** failureCount, 30000)
      }
    }
  )
  console.log('Success:', result)
} catch (error) {
  console.log('Failed after retries:', error)
}

```
暫停/繼續
```ts
class SimpleQueryWithPause {
  constructor() {
    this.state = {
      data: null,
      error: null,
      isLoading: false
    }
    this.continueFn = null
    this.online = true
    this.visible = true
    
    // 初始化時設置監聽器
    this.setupListeners()
  }

  setupListeners() {
    // 網路狀態監聽
    window.addEventListener('online', () => {
      this.setOnline(true)
      this.continue()  // 網路恢復時繼續
    })
    window.addEventListener('offline', () => {
      this.setOnline(false)
    })

    // 視窗焦點監聽
    window.addEventListener('visibilitychange', () => {
      this.setVisible(!document.hidden)
      if (!document.hidden) {
        this.continue()  // 視窗獲得焦點時繼續
      }
    })
  }

  setOnline(online) {
    this.online = online
    console.log('Network status:', online ? 'online' : 'offline')
  }

  setVisible(visible) {
    this.visible = visible
    console.log('Window visibility:', visible ? 'visible' : 'hidden')
  }

  canContinue() {
    return this.visible && this.online
  }

  pause() {
    return new Promise(resolve => {
      this.continueFn = value => {
        if (this.canContinue()) {
          resolve(value)
        }
      }
      console.log('Query paused')
    })
  }

  continue() {
    if (this.continueFn) {
      console.log('Query continuing')
      this.continueFn()
      this.continueFn = null
    }
  }

  async fetch(queryFn) {
    this.state.isLoading = true
    console.log('Starting query...')
    
    try {
      // 開始前檢查狀態
      if (!this.canContinue()) {
        console.log('Conditions not met, pausing...')
        await this.pause()
      }

      const data = await queryFn()
      
      this.state = {
        data,
        error: null,
        isLoading: false
      }
      
      console.log('Query completed successfully')
      return this.state
      
    } catch (error) {
      console.log('Query failed:', error)
      this.state = {
        data: null,
        error,
        isLoading: false
      }
      throw error
    }
  }
}

// 使用示例
async function example() {
  const query = new SimpleQueryWithPause()

  try {
    // 模擬 API 調用
    const result = await query.fetch(async () => {
      console.log('Fetching data...')
      await new Promise(resolve => setTimeout(resolve, 2000))
      return { message: 'Hello World' }
    })

    console.log('Result:', result)
  } catch (error) {
    console.error('Error:', error)
  }
}

// 運行示例
example()

// 模擬網路中斷和恢復
setTimeout(() => {
  window.dispatchEvent(new Event('offline'))
}, 1000)

setTimeout(() => {
  window.dispatchEvent(new Event('online'))
}, 3000)

// 模擬視窗切換
setTimeout(() => {
  document.hidden = true
  window.dispatchEvent(new Event('visibilitychange'))
}, 4000)

setTimeout(() => {
  document.hidden = false
  window.dispatchEvent(new Event('visibilitychange'))
}, 6000)

```

 訂閱狀態變化
```ts
// 基礎訂閱類別 - 實現訂閱者模式的核心邏輯
class Subscribable {
  constructor() {
    // 使用 Set 存儲監聽器，確保唯一性
    this.listeners = new Set()
    // 綁定 this 上下文，確保回調中的 this 指向正確
    this.subscribe = this.subscribe.bind(this)
  }

  // 提供訂閱機制，返回取消訂閱的函數
  subscribe(listener) {
    this.listeners.add(listener)
    // 返回取消訂閱函數，移除對應的監聽器
    return () => {
      this.listeners.delete(listener)
    }
  }

  // 檢查是否有活躍的監聽器
  hasListeners() {
    return this.listeners.size > 0
  }
}

// 查詢類 - 繼承訂閱功能，實現數據查詢和狀態管理
class SimpleQueryWithSubscribe extends Subscribable {
  constructor() {
    super()
    // 初始化查詢狀態
    this.state = {
      data: null,      // 查詢結果
      error: null,     // 錯誤信息
      isLoading: false // 加載狀態
    }
  }

  // 更新狀態並通知所有監聽器
  setState(newState) {
    // 更新狀態
    this.state = { ...this.state, ...newState }
    // 通知所有監聽器狀態變化
    this.listeners.forEach(listener => {
      listener(this.state)
    })
  }

  // 執行查詢的主要方法
  async fetch(queryFn) {
    // 設置加載狀態
    this.setState({ isLoading: true })
    
    try {
      // 執行查詢函數
      const data = await queryFn()
      // 更新成功狀態
      this.setState({
        data,
        error: null,
        isLoading: false
      })
      return this.state
      
    } catch (error) {
      // 更新錯誤狀態
      this.setState({
        data: null,
        error,
        isLoading: false
      })
      throw error
    }
  }
}

// 使用示例
const query = new SimpleQueryWithSubscribe()

// 訂閱狀態變化 - 每次狀態更新都會調用此回調
const unsubscribe = query.subscribe((state) => {
  console.log('Query state changed:', state)
})

// 執行查詢 - 觸發狀態變化並通知訂閱者
query.fetch(async () => {
  const response = await fetch('/api/data')
  return response.json()
})


```

定時更新資料
```ts
class SimpleQueryWithRefetch {
  constructor() {
    this.state = {
      data: null,
      error: null,
      isLoading: false
    }
    this.intervalId = null
  }

  startRefetch(options) {
    if (this.intervalId) {
      clearInterval(this.intervalId)
    }

    if (!options.refetchInterval) {
      return
    }

    this.intervalId = setInterval(() => {
 
     
      
      if (
     // 必要邏輯 : 由refetchIntervalInBackground控制，是否定時更新資料
    options.refetchIntervalInBackground
     // 非必要邏輯: 原始碼中，此處的判斷實際包含了 tab 的顯示狀態：
    // focusManager.isFocused() 會檢查 document?.visibilityState !== 'hidden'
    || document?.visibilityState !== 'hidden'
) {
        this.fetch()
      }
    }, options.refetchInterval)
  }

  async fetch(queryFn, options = {}) {
    try {
      const data = await queryFn()
      this.state = { data, error: null, isLoading: false }
      return this.state
    } catch (error) {
      this.state = { data: null, error, isLoading: false }
      throw error
    }
  }
}

// 步驟一：創建查詢實例
const query = new SimpleQueryWithRefetch()

// 步驟二：執行查詢並設置每1秒自動更新資料
query.fetch(
  () => fetch('/api/data').then(r => r.json()),
  {
    refetchInterval: 1000,
    refetchIntervalInBackground: true
  }
)
```

cacheTime
```ts
class SimpleQueryWithCache {
  constructor() {
    this.state = {
      data: null,
      error: null,
      isLoading: false
    }
    this.gcTimeout = null

    /* staleTime vs cacheTime 的資料處理差異：
     *
     * staleTime: 控制數據"過期"但還可以用（設定 5 天）
     * 過期前（0-5天）：直接用緩存數據
     * 過期後（5-7天）：先回傳舊數據 + 同時在背景更新
     * 
     * 就像牛奶的"最佳賞味期限"：
     * - 期限內（0-5天）：新鮮牛奶，直接喝
     * - 期限後（5-7天）：還能喝，但最好去買新鮮的
     * 
     * cacheTime: 控制數據完全"清除"（設定 7 天）
     * 過期前（0-7天）：數據保留在緩存中
     * 過期後（7天後）：數據被刪除，必須重新請求
     * 
     * 就像牛奶的"保存期限"：
     * - 期限內（0-7天）：放在冰箱還能喝
     * - 期限後（7天後）：必須倒掉，要買新的
     */

    this.cacheTime = 7 * 24 * 60 * 60 * 1000  // 7天
  }

  scheduleGc() {
    this.clearGc()
    
    // 設置緩存清理計時器
    // 7天後，數據會被完全清除
    this.gcTimeout = setTimeout(() => {
      this.remove()
    }, this.cacheTime)
  }

  clearGc() {
    if (this.gcTimeout) {
      clearTimeout(this.gcTimeout)
      this.gcTimeout = null
    }
  }

  remove() {
    // 完全清除數據，就像倒掉過期牛奶
    this.state = {
      data: null,
      error: null,
      isLoading: false
    }
  }

  async fetch(queryFn, options = {}) {
    try {
      const data = await queryFn()
      this.state = {
        data,
        error: null,
        isLoading: false
      }

      // 開始計算緩存的保存期限（7天）
      this.scheduleGc()
      
      return this.state
      
    } catch (error) {
      this.state = {
        data: null,
        error,
        isLoading: false
      }
      throw error
    }
  }

  destroy() {
    this.clearGc()
  }
}

// 使用示例
const query = new SimpleQueryWithCache()

// 使用 7 天的緩存時間
query.fetch(
  () => fetch('/api/data').then(r => r.json()),
  {
    cacheTime: 7 * 24 * 60 * 60 * 1000  // 7天後數據會被完全清除
  }
)
```

stale過期時，先回傳快取資料，同時背景更新為新資料
```ts
class SimpleQueryWithStale {
  constructor() {
    this.state = {
      data: null,
      fetchStatus: 'idle'  // 追蹤查詢狀態
    }
  }

  async fetch(queryFn) {
    // 檢查是否正在查詢中
    if (this.state.fetchStatus !== 'idle') {
      // 如果已有查詢在進行中，返回該查詢
      return this.currentPromise
    }

    this.state.fetchStatus = 'fetching'
    
    // 創建新查詢
    this.currentPromise = queryFn()
      .then(data => {
        this.state = {
          data,
          fetchStatus: 'idle'
        }
        return this.state
      })

    // 如果有舊數據，立即返回
    if (this.state.data) {
      return this.state
    }

    // 沒有舊數據，等待新查詢
    return this.currentPromise
  }
}

// 使用示例
const query = new SimpleQueryWithStale()

// 首次查詢：等待數據
const result1 = await query.fetch(() => 
  fetch('/api/todos').then(r => r.json())
)
console.log(result1)

// 第二次查詢：立即返回舊數據，背景更新
const result2 = await query.fetch(() => 
  fetch('/api/todos').then(r => r.json())
)
console.log(result2)





```

## mutation

精簡後的核心狀態機邏輯
```ts
// --- Mutation 類：核心狀態機的簡化實現  ---

// 為了簡化說明，這裡不會有「遇到錯誤時的重試邏輯」、「通知外部觀察者資料已更新」、「快取機制」等複雜邏輯，
// 只會專注於 mutation 的狀態變化和核心邏輯。

// 情境：
  // 假設你發送了一個訂單 API，這個訂單有三種狀態：
  // 1. 等待中 (pending)
  // 2. 成功 (success)
  // 3. 失敗 (error)
  // 當你發送訂單時，狀態會變成等待中 (pending)，
  // 然後根據 API 的回應，狀態會變成成功 (success) 或失敗 (error)。



class Mutation {

  // _dispatch 是內部方法，用於觸發狀態更新。
  _dispatch(action) {
    // 定義function : 計算出新的狀態。
    const reducer = (currentState, currentAction) => {
      switch (currentAction.type) {
        case 'pending':
          // 當 mutation 執行時，狀態會改為 'pending'。
          return {
            ...currentState, 
            status: 'pending',
            data: undefined,
            // ...省略其他狀態屬性
          };
        case 'success':
          return {
            ...currentState,
            status: 'success',
            data: currentAction.data, 
            // ...省略其他狀態屬性
          };
        case 'error':
          return {
            ...currentState,
            status: 'error',
            error: currentAction.error, 
            data: undefined,
            // ...省略其他狀態屬性
          };
        default:
          // 如果接收到未知的 action type，保持當前狀態不變。
          return currentState;
      }
    };

    // 呼叫 reducer 計算新狀態，並更新 this.state。
    this.state = reducer(this.state, action);

    // 在完整的 TanStack Query 中，狀態更新後，會在這裡通知所有觀察者 (MutationObserver)，
    // 讓 UI 層或其他訂閱者能夠響應狀態變化。例如：
    // this._observers.forEach(observer => observer.onMutationUpdate(action));
    // 這裡暫時省略，以專注核心狀態機。
  }

  // execute 方法是啟動 mutation 執行的入口。
  // 它接收執行 mutation 所需的變量 (variables)。
  async execute(variables) {
    // 步驟 1: 將狀態設置為 'pending'，預設為 'idle'，表示 mutation 尚未開始。
    // 表示 mutation 操作已開始，正在等待異步函數 (mutationFn) 的結果。
    this._dispatch({ type: 'pending', variables: variables });

    try {
      // 步驟 2: 執行用戶提供的核心異步函數 (mutationFn)
      if (!this.options.mutationFn) {
        // 必須提供 mutationFn，否則不知道要執行什麼操作。
        throw new Error('No mutationFn provided in options');
      }

      // 直接呼叫 mutationFn。
      // 在完整的 TanStack Query 中，這裡並非直接呼叫，而是通過一個名為 `Retryer` 的組件。
      // `Retryer` 負責實際執行 `mutationFn`，並且能夠處理網絡錯誤時的自動重試、
      // 判斷網絡狀態（在線/離線）來決定是否暫停或繼續執行。
      // 這種委派使得 Mutation 類本身不必關心複雜的重試和網絡邏輯。
      // 為簡化，此處直接 await，假設 mutationFn 只執行一次且不處理重試。
      const data = await this.options.mutationFn(variables);

      // 步驟 3: 處理 mutationFn 成功的情況
      // 在完整的 TanStack Query 中，這裡會先呼叫用戶配置的 onSuccess 回呼函數，
      // 允許用戶在 mutation 成功後執行一些副作用，例如：使相關的查詢快取失效、顯示成功提示等。
      // if (this.options.onSuccess) {
      //   await this.options.onSuccess(data, variables, this.state.context); 
      // }

      // 步驟 4: 將狀態設置為 'success'
      this._dispatch({ type: 'success', data: data });

      return data; // 返回成功數據
    } catch (error) {

      // 步驟 5: 處理 mutationFn 失敗的情況
      // 在完整的 TanStack Query 中，這裡會先呼叫用戶配置的 onError 回呼函數，
      // 允許用戶處理錯誤，例如：記錄錯誤、顯示錯誤提示、執行回滾操作等。
      // if (this.options.onError) {
      //   await this.options.onError(error, variables, this.state.context); 
      // }

      // 步驟 6: 將狀態設置為 'error'
      this._dispatch({ type: 'error', error: error });

      throw error; // 重新拋出錯誤，以便 execute 的呼叫者可以捕獲和處理
    } finally {
      // 在完整的 TanStack Query 中，`finally` 塊通常會執行一些清理工作，
      // 或者通知 `MutationCache` 當前 mutation 已完成（無論成功或失敗），
      // 以便 `MutationCache` 可以決定是否執行隊列中的下一個 mutation（如果配置了序列化執行）。
      // 例如：this._mutationCache.runNext(this);
      // 如果沒有這個機制，同時觸發的多個 mutation 可能會並發執行，
      // 而 `runNext` 提供了控制併發或序列化執行的能力。
      // 在此最簡核心版本中，我們假設 mutation 是獨立執行的，故省略此邏輯。
    }
  }
}

```


這樣的簡化寫法是符合原始碼的最核心邏輯嗎? 我只需要檢查最終的邏輯判斷思考方向是否一致，有一些邏輯簡化複雜性是可以接受的

基於原始碼的寫法，在最最最基礎的例子加入簡化過的 "重試機制" 邏輯 ，不需要列出不相關的邏輯進來，讓我更好理解

以開發者的角度來說明，還有哪些功能可以再逐步加入說明的? 先從相關性最高的列出
