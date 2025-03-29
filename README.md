# Pinia

## 核心概念
## Store
### 為什麼 `defineStore` 的第一個參數需要是唯一的 Store ID？

```ts
export const useAlertsStore = defineStore('alerts', { ... })
```
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
----
### Option Store 和 Setup Store 實現的邏輯有什麼差別?

#### 都是通過同一個函數 createSetupStore 來處理

##### 處理流程
defineStore 會根據第二個參數的類型來決定創建哪種 store：
- 如果是函數，創建 Setup Store
- 如果是物件，創建 Options Stor
```ts
export function defineStore(
  id: any,
  setup?: any,
  setupOptions?: any
): StoreDefinition {
  let options
  const isSetupStore = typeof setup === 'function'
  options = isSetupStore ? setupOptions : setup

  function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
    // ...
    if (!pinia._s.has(id)) {
      // 根據 store 類型選擇創建方式
      if (isSetupStore) {
        createSetupStore(id, setup, options, pinia)
      } else {
        createOptionsStore(id, options as any, pinia)
      }
    }
    // ...
  }
  return useStore
}
```

Options Store 轉換為Setup Store的過程

```ts
function createOptionsStore<Id extends string>(
  id: Id,
  options: DefineStoreOptions<Id, S, G, A>,
  pinia: Pinia,
  hot?: boolean
): Store<Id, S, G, A> {
  const { state, actions, getters } = options
  
  function setup() {
    // 將 options 格式轉換為 setup 格式
    const localState = toRefs(pinia.state.value[id])
    
    return assign(
      localState,
      actions,
      Object.keys(getters || {}).reduce((computedGetters, name) => {
        // 轉換 getters 為 computed
        computedGetters[name] = markRaw(
          computed(() => {
            const store = pinia._s.get(id)!
            return getters![name].call(store, store)
          })
        )
        return computedGetters
      }, {})
    )
  }

  // 最終還是調用 createSetupStore
  store = createSetupStore(id, setup, options, pinia, hot, true)
  
  return store as any
}
```

```ts
function createSetupStore<Id extends string>(
  $id: Id,
  setup: () => SS,
  options: DefineSetupStoreOptions<Id, S, G, A>,
  pinia: Pinia,
  hot?: boolean,
  isOptionsStore?: boolean
): Store<Id, S, G, A> {
  // ... 
  
  // 執行 setup 函數
  const setupStore = pinia._e.run(() =>
    /*
      使用 effectScope() 創建獨立的響應式作用域
      這個作用域用於管理 store 中所有的響應式效果（computed、watch 等）
      當 store 被銷毀時，可以一次性清理所有效果
     */
    (scope = effectScope()).run(() => setup())
  )!

  // 處理 setup 返回的內容
  for (const key in setupStore) {
    const prop = setupStore[key]

    if ((isRef(prop) && !isComputed(prop)) || isReactive(prop)) {
      // 處理 state
    } else if (typeof prop === 'function') {
      // 處理 actions
    } else if (isComputed(prop)) {
      // 處理 getters
    }
  }
  
  // ...
  return store
}
```
### Store中可以使用inject()來獲取全域依賴？

要在 Store 中安全地使用 inject()，需要注意以下關鍵步驟：

1. 首先在應用入口提供依賴
```ts
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import { router } from './router'
import { i18n } from './i18n'
import App from './App.vue'

const app = createApp(App)

// 先提供全域依賴
app.provide('router', router)
app.provide('i18n', i18n)

// 再初始化 pinia
const pinia = createPinia()
app.use(pinia)

app.mount('#app')
```

2. 然後在 Store 中使用 inject
```ts
import { inject } from 'vue'
import { defineStore } from 'pinia'
import type { Router } from 'vue-router'
import type { I18n } from 'vue-i18n'

export const useMainStore = defineStore('main', () => {
  const router = inject<Router>('router')
  const i18n = inject<I18n>('i18n')
  
  // 確保依賴已被正確注入
  if (!router || !i18n) {
    throw new Error('Router or i18n instance not provided')
  }
  
  // 在 store 中使用這些注入的依賴
  function navigate(path: string) {
    router.push(path)
  }
  
  function translate(key: string) {
    return i18n.t(key)
  }
  
  return {
    navigate,
    translate
  }
})
```
### 怎麼做到的?
#### Pinia實例的創建與注入
當應用調用app.use(pinia)時：
 - Pinia通過app.provide()將自己注入到應用的全局上下文
 - 這使得所有組件和store都能訪問到pinia實例
```ts
// packages/pinia/src/createPinia.ts
export function createPinia(): Pinia {
  const scope = effectScope(true)  // 創建一個全局的effect作用域
  
  const pinia = markRaw({
    install(app: App) {
      // 關鍵點1: 將pinia實例注入到整個應用
      app.provide(piniaSymbol, pinia)
      pinia._a = app
      // ...
    },
    _e: scope,  // 保存effect作用域
    // ...
  })
  return pinia
}
```

#### Store使用注入的上下文
當創建store時：
 - Store通過inject()獲取pinia實例
 - 因為store在應用的provide/inject層級中執行
 - 所以可以訪問到所有通過app.provide()提供的全域屬性
```ts
// packages/pinia/src/store.ts
function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
  const hasContext = hasInjectionContext()
  // 關鍵點2: 從當前執行上下文獲取已注入的pinia實例
  pinia = (hasContext ? inject(piniaSymbol, null) : null)
  // ...
}
```

---

### 為什麼不能對 store/reactive 進行解構
```ts
<script setup>
import { useCounterStore } from '@/stores/counter'
import { computed } from 'vue'

const store = useCounterStore()
// ❌ 这将不起作用，因为它破坏了响应性
// 这就和直接解构 `props` 一样
const { name, doubleCount } = store
name // 将始终是 "Eduardo" //

//  ✅ 需要使用 storeToRefs()。它将为每一个响应式属性创建引用
//  💡 `name` 和 `doubleCount` 是响应式的 ref
//  💡 同时通过插件添加的属性也会被提取为 ref
const { name, doubleCount } = storeToRefs(store)
//  ✅  作为 action 的 increment 可以直接解构
const { increment } = store

</script>
```


#### 原因

store 實際上是 reactive 物件，當對它進行解構時，會破壞響應性。這是因為 JavaScript 的解構賦值是基於值的複製，而不是引用。

直接解構的問題:
```ts
const store = reactive({
  count: 0
})

// ❌ 解構賦值 - 只會得到當下的值 0
const { count } = store
console.log(count) // 0

store.count++ 
console.log(count) // 仍然是 0，因為 count 只是一個普通的變量，不再與 store.count 有關聯
```

使用 toRef 改寫:
```ts
// ✅ 使用 toRef - 創建了一個響應式引用
const count = toRef(store, 'count')
console.log(count.value) // 0

store.count++
console.log(count.value) // 1，因為 count 是一個 ref，與原始數據保持連接
```

最後再看看 storeToRefs 的實現來理解這一點：
- 只會處理兩種類型的屬性
 - 有 effect 屬性的 computed/getters
 - 是 ref 或 reactive 的響應式數據
- Actions是普通的函數，不符合上述兩種情況，所以在 storeToRefs回傳值是拿不到 actions的。
```ts
export function storeToRefs<SS extends StoreGeneric>(store: SS) {
  // 1. 先使用 toRaw 獲取原始 store 對象
  const rawStore = toRaw(store)
  // 建立一個新的空對象來存儲 refs
  const refs = {}

  // 2. loop原始 store 的所有屬性
  for (const key in rawStore) {
    const value = rawStore[key]
    
    // 3. 如果屬性有 effect（表示是 computed/getter），創建新的 computed ref
    if (value.effect) {
      refs[key] = computed({
        get: () => store[key],
        set(value) {
          store[key] = value
        }
      })
    } 
    // 4. 如果屬性是 ref 或 reactive 對象，使用 toRef 創建引用
    else if (isRef(value) || isReactive(value)) {
      refs[key] = toRef(store, key)
    }

    // 5. 其他類型的屬性（如 actions/methods）則被忽略
  }

  // 回傳所有引用的對象
  return refs
}
```
----

## State

### Option Store的$reset()是怎麼實現的?

```ts
const $reset = isOptionsStore
  ? function $reset() {
      /*
        options是defineStore傳入的Object

        defineStore('store', {
          state: () => ({ count: 0 })
        })
       */
      const { state } = options

      // 會重置到初始狀態
      const newState = state ? state() : {}
      /*
        使用 $patch 來更新狀態。$patch 的特別之處在於它會：
          - 暫時停止監聽狀態變化（isListening = false）
          - 將所有更改集中處理
          - 最後才一次性通知所有訂閱者，避免觸發多次更新
          - 這樣可以提升效能並保持狀態更新的一致性
       */
      this.$patch(($state) => {
        assign($state, newState)
      })
    }
  : /* istanbul ignore next */
    __DEV__
    ? () => {
        // setup store 預設不支援 reset 功能，需要自己實作
        throw new Error(
          `🍍: Store "${$id}" is built using the setup syntax and does not implement $reset().`
        )
      }
    : noop
```
---

### mapState是怎麼實現的?

mapState主要有兩種使用方式，根據傳入參數的不同會有不同實現:

1.陣列:
```ts
mapState(useCounterStore, ['count'])
```

2.物件:
```ts
mapState(useCounterStore, {
  myOwnName: 'count',
  double: store => store.count * 2,
  withThis(store) {
    return store.count + this.double
  }
})
```

- 若傳入陣列，會遍歷陣列並為每個key生成一個computed property
- 若傳入物件，會遍歷物件的key，針對不同value類型:
   - 若是字串: 直接返回store對應的值
   - 若是函數: 執行該函數並傳入store實例

```ts
export function mapState(useStore, keysOrMapper) {
  return Array.isArray(keysOrMapper)
    ? keysOrMapper.reduce((reduced, key) => {
        // 陣列形式: 直接返回store[key]
        reduced[key] = function() {
          return useStore(this.$pinia)[key]
        }
        return reduced
      }, {})
    : Object.keys(keysOrMapper).reduce((reduced, key) => {
        // 物件形式
        reduced[key] = function() {
          const store = useStore(this.$pinia)
          const storeKey = keysOrMapper[key]
          return typeof storeKey === 'function'
            // 如果是函數則調用並傳入store
            // this是指向store的實例
            ? storeKey.call(this, store) 
            // 如果是字串則直接返回store的值
            : store[storeKey]
        }
        return reduced
      }, {})
}
```
