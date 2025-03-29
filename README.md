# Pinia

## 核心概念
### 為什麼 `defineStore` 的第一個參數需要是唯一的 Store ID？

```ts
export const useAlertsStore = defineStore('alerts', { ... })
```
----

#### Pinia 使用 Map 來儲存所有 store  
Pinia 內部使用一個 `Map<string, StoreGeneric>` 結構：
```ts
_s: new Map<string, StoreGeneric>()
```
該 Map 以 store ID 為 key，對應 store 實例。

#### Store 建立與檢查流程  
在 `defineStore()` 內部，每次調用 store 時：
- **檢查 ID 是否已存在**
- **如果 ID 不存在，則創建新的 store**
- **如果 ID 已存在，則返回已存在的 store**
  
```ts
if (!pinia._s.has(id)) {
  // 創建新的 store
  ....
}

const store: StoreGeneric = pinia._s.get(id)!
```

#### Store 實例共享範例  
```ts
// 定義 store
export const useAlertsStore = defineStore('alerts', { ... })

// 在不同的組件中使用
const alertsStore1 = useAlertsStore()
const alertsStore2 = useAlertsStore()

// alertsStore1 === alertsStore2  // true，指向同一個實例
```

