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
