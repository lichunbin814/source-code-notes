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

---

### $patch 和 $state 賦值有什麼差別？

官網範例
```ts
// 这实际上并没有替换`$state`
store.$state = { count: 24 }
// 在它内部调用 `$patch()`：
store.$patch({ count: 24 })
```

#### 實作差異

#### 在 Pinia 中，雖然 $state 賦值最終也會調用 $patch，但兩者在實作上有重要的差異：
$state賦值會被轉換為 $patch 的函式形式，使用 assign 進行合併
``` ts
store.$state = { count: 24 }：

// store.ts
Object.defineProperty(store, '$state', {
  set: (state) => {
    $patch(($state) => {
      assign($state, state)  // 直接使用 Object.assign
    })
  },
})
```

$patch直接使用 mergeReactiveObjects 進行合併
```ts
store.$patch({ count: 24 });


// store.ts
function $patch(
  partialStateOrMutator:
    | _DeepPartial<UnwrapRef<S>>
    | ((state: UnwrapRef<S>) => void)
): void {
  if (typeof partialStateOrMutator === 'function') {
    partialStateOrMutator(pinia.state.value[$id])
  } else {
    mergeReactiveObjects(pinia.state.value[$id], partialStateOrMutator)
  }
}

// mergeReactiveObjects 的實作
function mergeReactiveObjects<T>(target: T, patchToApply: _DeepPartial<T>): T {
  // 專門處理 Map 類型的合併
  // 如果目標和補丁都是 Map 實例，則遍歷補丁的所有項目並設置到目標 Map
  if (target instanceof Map && patchToApply instanceof Map) {
    patchToApply.forEach((value, key) => target.set(key, value))
  } 
  // 專門處理 Set 類型的合併
  // 如果目標和補丁都是 Set 實例，則將補丁的所有項目添加到目標 Set
  else if (target instanceof Set && patchToApply instanceof Set) {
    patchToApply.forEach(target.add, target)
  }

  // 遍歷補丁對象的所有屬性
  for (const key in patchToApply) {
    // 跳過原型鏈上的屬性，只處理對象自身的屬性
    if (!patchToApply.hasOwnProperty(key)) continue
    
    // 獲取補丁中的新值
    const subPatch = patchToApply[key]
    // 獲取目標對象中的當前值
    const targetValue = target[key]
    
    // 判斷是否需要深度合併
    // 條件說明：
    // 1. targetValue 是普通對象 (不是陣列、函式等)
    // 2. subPatch 也是普通對象
    // 3. 目標對象本身具有這個屬性
    // 4. subPatch 不是 ref (避免破壞響應式)
    // 5. subPatch 不是 reactive 對象 (避免重複處理響應式對象)
    if (
      isPlainObject(targetValue) &&
      isPlainObject(subPatch) &&
      target.hasOwnProperty(key) &&
      !isRef(subPatch) &&
      !isReactive(subPatch)
    ) {
      // 遞迴合併巢狀對象
      target[key] = mergeReactiveObjects(targetValue, subPatch)
    } else {
      // 如果不滿足深度合併的條件，直接替換值
      // 這包括：
      // - 基本類型值
      // - 陣列
      // - 新屬性 (目標對象中不存在的)
      // - ref 或 reactive 對象
      target[key] = subPatch
    }
  }

  return target
}
```

#### $patch 和 $state 賦值的合併差異


```ts
1.基本物件更新
const store = defineStore('main', {
  state: () => ({
    count: 0,
    name: 'test'
  })
})

// 使用 $state
store.$state = { count: 24 }
console.log(store.$state)
// 結果: { count: 24, name: 'test' }
// name 屬性被保留，因為使用 Object.assign 合併

// 使用 $patch
store.$patch({ count: 24 })
console.log(store.$state)
// 結果: { count: 24, name: 'test' }
// 效果相同，因為都是淺層更新

2.巢狀物件更新
const store = defineStore('main', {
  state: () => ({
    user: {
      name: 'John',
      settings: {
        theme: 'dark',
        notifications: true
      }
    }
  })
})

// 使用 $state
store.$state = { 
  user: { 
    settings: { theme: 'light' } 
  } 
}
console.log(store.$state)
// 結果: { 
//   user: { 
//     settings: { theme: 'light' }  // notifications 被覆蓋掉了
//   } 
// }
// name 也被覆蓋掉了，因為 Object.assign 是淺合併

// 使用 $patch
store.$patch({ 
  user: { 
    settings: { theme: 'light' } 
  } 
})
console.log(store.$state)
// 結果: { 
//   user: { 
//     name: 'John',  // 保留原有值
//     settings: { 
//       theme: 'light',  // 更新
//       notifications: true  // 保留原有值
//     } 
//   } 
// }
// mergeReactiveObjects 進行深度合併，保留未更新的屬性

3.特殊類型更新 (Map)
const store = defineStore('main', {
  state: () => ({
    preferences: new Map([
      ['theme', 'dark'],
      ['language', 'en']
    ])
  })
})

// 使用 $state
store.$state = { 
  preferences: new Map([['theme', 'light']]) 
}
console.log(store.$state.preferences)
// 結果: Map { 'theme' => 'light', 'language' => 'en' }
// language 設定會被保留,因為 Object.assign 會合併物件

// 使用 $patch
store.$patch({ 
  preferences: new Map([['theme', 'light']]) 
})
console.log(store.$state.preferences)
// 結果: Map { 'theme' => 'light', 'language' => 'en' }
// mergeReactiveObjects 會正確處理 Map 的合併

// 兩者實際上有相同的結果,因為:
// 1. $state 使用 Object.assign 合併,會保留原有屬性
// 2. $patch 使用 mergeReactiveObjects,對於 Map 類型會逐一設置 entries

4.多層級陣列更新
const store = defineStore('main', {
  state: () => ({
    items: [
      { id: 1, data: { value: 'old' } },
      { id: 2, data: { value: 'test' } }
    ]
  })
})

// 使用 $state
store.$state = { 
  items: [{ id: 1, data: { value: 'new' } }] 
}
console.log(store.$state)
// 結果: { 
//   items: [{ id: 1, data: { value: 'new' } }] 
// }
// 整個陣列被替換，id: 2 的項目消失

// 使用 $patch
store.$patch({ 
  items: [{ id: 1, data: { value: 'new' } }] 
})
console.log(store.$state)
// 與$state結果相同
// 結果: { 
//   items: [{ id: 1, data: { value: 'new' } }] 
// }
// 整個陣列被替換，id: 2 的項目消失

5.直接修改深層屬性：
// $patch
store.$patch((state) => {
  state.items[0].data.value = 'new'
})

// 直接修改 $state
store.$state.items[0].data.value = 'new'  // 這也是可行的
```

---

### 用watch或$subscribe監聽state時，callback的執行時機有差異嗎?

watch 是 Vue 的響應式系統功能:
 - 監聽整個狀態物件的變化
 - 不論是單個還是多個屬性變化,只要是在同一個tick內發生,都只會觸發一次callback
 - 監聽時機取決於 watch 選項的 flush 設定:
   -- 默認是 'pre' (組件更新前)
   -- 可以設置 'post' (組件更新後)
   -- 可以設置 'sync' (同步執行)

$subscribe 是 Pinia 特有的功能:
 - 在內部使用 watch 實現
 - 有更細緻的控制機制，可以區分同步和非同步的監聽狀態
 - 在執行 $patch 時會暫時關閉監聽，等待所有更改完成後才觸發callback

#### watch

```ts
watch(
  pinia.state,
  (state) => {
    localStorage.setItem('piniaState', JSON.stringify(state))
  },
  { deep: true }
)

store.count++
store.name = 'new name'

// 這些修改會遵循 Vue 的響應式系統更新機制，在下一個更新週期觸發 callback
```

#### 使用$subscribe
```ts

store.$subscribe((mutation, state) => {
  localStorage.setItem('piniaState', JSON.stringify(state))
})

// 當使用 $patch 進行多個更改時：
store.$patch({
  count: store.count + 1,
  name: 'new name'
})

// $subscribe 的回調會在所有更改完成後，下一個更新週期觸發
// 在內部實現中會暫時關閉並在適當時機恢復監聽：
function $patch() {
  // 暫時關閉監聽
  isListening = isSyncListening = false  
  
  // ... 執行所有更改 ...
  
  // 觸發訂閱者
  triggerSubscriptions(
    subscriptions,
    subscriptionMutation,
    pinia.state.value[$id]
  )
}

```

----

### $patch的修改會觸發$subscribe內部watch的callback嗎？
$patch 修改 state.value 時，watch 會偵測到變化，但 callback 不會執行，因為被 isListening 的條件阻擋了
 - 因為$patch 執行時會暫時關閉監聽，避免修改過程中觸發 watch
 - 在修改完成後，$patch 會手動調用 triggerSubscriptions 來觸發訂閱
 - 這種設計可以確保所有修改完成後才觸發一次回調，並提供完整的修改信息

#### 原理
$patch 通過以下機制來控制訂閱的觸發:
 - 首先暫時關閉監聽(isListening = false)
 - 使用 mergeReactiveObjects 修改 pinia.state.value[$id]
 - 修改完成後，手動調用 triggerSubscriptions 觸發訂閱
 - 最後在下一個 tick 恢復監聽


```ts
function $patch(
  partialStateOrMutator:
    | _DeepPartial<UnwrapRef<S>>
    | ((state: UnwrapRef<S>) => void)
): void {
  let subscriptionMutation: SubscriptionCallbackMutation<S>
  isListening = isSyncListening = false  // 關閉監聽
  
  if (typeof partialStateOrMutator === 'function') {
    partialStateOrMutator(pinia.state.value[$id] as UnwrapRef<S>)
  } else {
    mergeReactiveObjects(pinia.state.value[$id], partialStateOrMutator)
  }

  // 手動觸發訂閱
  triggerSubscriptions(
    subscriptions,
    subscriptionMutation,
    pinia.state.value[$id]
  )

  // 下一個 tick 恢復監聽
  nextTick().then(() => {
    if (activeListener === myListenerId) {
      isListening = true
    }
  })
}
```

---

### $subscribe和watch的底層監聽邏輯差異

#### 關鍵差異

觸發控制機制：
 - watch: 純粹依賴 Vue 的響應式系統，當響應式數據變化時就會觸發，可通過 flush 選項控制觸發時機
 - subscribe: 通過 isListening/isSyncListening 標誌來增加額外的控制層，決定是否觸發回調
```ts
// store.ts
$subscribe(callback, options = {}) {
  const stopWatcher = scope.run(() =>
    watch(
      () => pinia.state.value[$id] as UnwrapRef<S>,
      (state) => {
        // 關鍵：這裡增加了控制層
        if (options.flush === 'sync' ? isSyncListening : isListening) {
          callback({
            storeId: $id,
            type: MutationType.direct,
            events: debuggerEvents as DebuggerEvent,
          }, state)
        }
      },
      assign({}, $subscribeOptions, options)
    )
  )!
}

// 相比之下，直接使用 watch 時：
watch(store, (newValue, oldValue) => {
  // 依賴 Vue 的響應式系統和 flush 選項
  callback(newValue, oldValue)
})
```

實現方式：
 - watch: 直接使用 Vue 的 watch API，完全依賴 Vue 的響應式系統
 - subscribe:
  -- 內部也使用了 watch   

```ts
// store.ts - subscribe 的實現
$subscribe(callback, options = {}) {
  // 1. 創建監聽
  const stopWatcher = scope.run(() =>
    watch(
      () => pinia.state.value[$id],
      (state) => {
        if (options.flush === 'sync' ? isSyncListening : isListening) {
          callback(/*...*/)
        }
      },
      options
    )
  )!
  
  // 2. 註冊訂閱
  const removeSubscription = addSubscription(
    subscriptions,
    callback,
    options.detached,
    // 清理函數：當訂閱被移除時，停止 watch
    () => stopWatcher()  
  )
  
  return removeSubscription
}

// subscriptions.ts - 訂閱相關的工具函數
export function addSubscription<T extends _Method>(
  subscriptions: T[],
  callback: T,
  detached?: boolean,
  onCleanup: () => void = noop
) {
  // 將callback加入訂閱陣列
  subscriptions.push(callback)
  
  // 返回移除訂閱的函數
  const removeSubscription = () => {
    const idx = subscriptions.indexOf(callback)
    if (idx > -1) {
      subscriptions.splice(idx, 1)
      onCleanup()
    }
  }
  
  if (!detached && getCurrentScope()) {
    // 組件卸載時自動清理訂閱
    onScopeDispose(removeSubscription)
  }
  
  return removeSubscription
}
```

觸發時機：
 - watch: 響應式數據變化時立即觸發，具體時機由 flush 選項控制
 - subscribe:
  -- 受 isListening/isSyncListening 控制
  -- 在 $patch 中會先禁用監聽，等所有修改完成後再一次性觸發

```ts
// store.ts - $patch 中的控制
function $patch() {
  // 1. 開始修改前，關閉所有監聽
  isListening = isSyncListening = false
  
  // 2. 執行修改
  if (typeof partialStateOrMutator === 'function') {
    partialStateOrMutator(pinia.state.value[$id])
  } else {
    mergeReactiveObjects(pinia.state.value[$id], partialStateOrMutator)
  }
  
  // 3. 控制監聽重啟的時機
  const myListenerId = (activeListener = Symbol())
  
  // 異步訂閱在下一個 tick 恢復
  nextTick().then(() => {
    if (activeListener === myListenerId) {
      isListening = true  // 恢復異步監聽
    }
  })
  
  // 同步訂閱立即恢復
  isSyncListening = true
  
  // 4. 手動觸發訂閱
  triggerSubscriptions(
    subscriptions,
    subscriptionMutation,
    pinia.state.value[$id]
  )
}

// subscriptions.ts - 手動觸發訂閱的實現
export function triggerSubscriptions<T extends _Method>(
  subscriptions: T[],
  ...args: Parameters<T>
) {
  // 先複製陣列再遍歷，避免在觸發過程中訂閱列表被修改影響遍歷
  subscriptions.slice().forEach((callback) => {
    callback(...args)
  })
}
```

批次處理的差異：
  - watch: 每個響應式屬性的變化都會獨立觸發，透過 flush 選項控制觸發時機 ('pre'/'post'/'sync')
  - subscribe: 在triggerSubscriptions觸發的 watch callback的加上增加了控制層
   -- isListening：控制異步訂閱（flush: 'post'/'pre'）
   -- isSyncListening：控制同步訂閱（flush: 'sync'）

```ts
// store.ts
// $subscribe 中的監聽callback
$subscribe(callback, options = {}) {
  ....
  watch(
    () => pinia.state.value[$id],
    (state) => {
      // 這裡控制是否能觸發callback
      if (options.flush === 'sync' ? isSyncListening : isListening) {
        callback(/*...*/)
      }
    }
  )
  ...
}
// $patch 中的控制
function $patch() {
  // 1. 先禁用所有監聽
  isListening = isSyncListening = false
  
  // 2. 執行修改
  if (typeof partialStateOrMutator === 'function') {
    partialStateOrMutator(pinia.state.value[$id])
  } else {
    mergeReactiveObjects(pinia.state.value[$id], partialStateOrMutator)
  }
  
  // 3. 設置不同的監聽狀態
  const myListenerId = (activeListener = Symbol())
  nextTick().then(() => {
    if (activeListener === myListenerId) {
      isListening = true  // 異步監聽
    }
  })
  isSyncListening = true  // 同步監聽
  
  // 4. 手動觸發
  triggerSubscriptions(subscriptions, subscriptionMutation, pinia.state.value[$id])
}

export function triggerSubscriptions<T extends _Method>(
  subscriptions: T[],
  ...args: Parameters<T>
) {
  // 先複製陣列再遍歷，避免在觸發過程中訂閱列表被修改影響遍歷
  subscriptions.slice().forEach((callback) => {
    callback(...args)
  })
}

// test case
it('works with multiple different flush', async () => {
  s1.user = 'Edu'
  expect(spyPre).toHaveBeenCalledTimes(0)    // 異步，等待 nextTick
  expect(spyPost).toHaveBeenCalledTimes(0)   // 異步，等待 nextTick
  expect(spySync).toHaveBeenCalledTimes(1)   // 同步，立即執行
  await nextTick()
  expect(spyPre).toHaveBeenCalledTimes(1)    // nextTick 後執行
  expect(spyPost).toHaveBeenCalledTimes(1)   // nextTick 後執行
  expect(spySync).toHaveBeenCalledTimes(1)   // 保持不變
})
```

---



