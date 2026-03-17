**Vue 3 的组合式函数（Composables）**—— 也就是 Vue 生态中常说的 “Vue Hooks”。它是 Vue 3 基于 Composition API 提供的逻辑复用方案，完美解决了 Vue 2 中 Options API 和 mixins 的痛点。

## 一、为什么需要 Composables？

在 Vue 2 中，我们用 **Options API**（`data`/`methods`/`computed`/`watch`/ 生命周期）写组件，存在两个明显问题：

1. **逻辑分散**：同一个功能的代码（如 “鼠标位置跟踪”）会被拆分到 `data`、`methods`、`mounted`、`beforeUnmount` 中，组件大了难以维护。
2. **复用困难**：用 `mixins` 复用逻辑时，会有**命名冲突**、**来源不明**的问题（不知道某个数据 / 方法来自哪个 mixin）。

**Composables** 的出现彻底解决了这些问题：它把相关逻辑封装在一个独立的函数中，既可以在组件内直接使用，也可以跨组件复用，代码更清晰、更灵活。

## 二、Vue 3 核心响应式 API（基础）

在写 Composables 之前，我们需要先掌握 Vue 3 最核心的几个响应式 API：

### 1. `ref`：定义基本类型的响应式数据

用于 `string`/`number`/`boolean` 等基本类型，返回一个对象，值存在 `.value` 中（模板中自动解包，无需写 `.value`）。

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0) // 初始值为 0
const increment = () => count.value++ // JS 中需通过 .value 访问
</script>

<template>
  <p>计数：{{ count }}</p> <!-- 模板中自动解包 -->
  <button @click="increment">+1</button>
</template>
```

### 2. `reactive`：定义对象类型的响应式数据

用于 `object`/`array` 等引用类型，直接返回响应式对象，无需 `.value`。

```vue
<script setup>
import { reactive } from 'vue'

const user = reactive({ name: '张三', age: 25 })
const updateName = () => user.name = '李四' // 直接修改属性
</script>
```

### 3. `computed`：计算属性（带缓存）

基于响应式数据派生新值，**只有依赖变化时才重新计算**，否则直接返回缓存结果（性能优化利器）。

```vue
<script setup>
import { ref, computed } from 'vue'

const firstName = ref('张')
const lastName = ref('三')

// 计算全名，只有 firstName 或 lastName 变化时才重新计算
const fullName = computed(() => `${firstName.value}${lastName.value}`)
</script>
```

### 4. `watch`/`watchEffect`：监听数据变化

#### `watch`：明确指定监听对象

- 可以获取**旧值**和**新值**。
- 默认**懒执行**（首次加载不执行，只有数据变化时才执行）。

```vue
<script setup>
import { ref, watch } from 'vue'

const count = ref(0)

// 监听 count，获取旧值和新值
watch(count, (newVal, oldVal) => {
  console.log(`count 变化：${oldVal} → ${newVal}`)
})
</script>
```

#### `watchEffect`：自动收集依赖

- 自动追踪回调中用到的响应式数据，无需明确指定监听对象。
- **立即执行**（首次加载就执行一次）。

```vue
<script setup>
import { ref, watchEffect } from 'vue'

const count = ref(0)

// 自动收集 count 为依赖，count 变化时自动执行
watchEffect(() => {
  console.log(`当前 count：${count.value}`)
})
</script>
```

## 三、Vue 3 生命周期 Hooks

Vue 3 的生命周期钩子以 `onXxx` 形式命名，需从 `vue` 中引入，直接在 `setup` 中调用即可：

|Vue 3 钩子|触发时机|常用场景|
|---|---|---|
|`onMounted`|组件挂载到 DOM **之后**（⭐️最常用）|操作 DOM、发起 API 请求、初始化第三方库|
|`onUpdated`|组件数据变化，DOM 更新 **之后**|操作更新后的 DOM|
|`onBeforeUnmount`|组件卸载 **之前**（⭐️常用）|清理定时器、取消事件监听、销毁第三方实例（避免内存泄漏）|
|`onUnmounted`|组件卸载 **之后**|极少使用|
|`onActivated`|被 `<keep-alive>` 缓存的组件**激活**时|恢复播放、重新请求数据|
|`onDeactivated`|被 `<keep-alive>` 缓存的组件**停用时**|暂停播放、保存草稿|

**示例**：

```vue
<script setup>
import { onMounted, onBeforeUnmount } from 'vue'

onMounted(() => {
  console.log('组件挂载完成，可以操作 DOM 了')
})

onBeforeUnmount(() => {
  console.log('组件即将卸载，清理资源')
})
</script>
```

## 四、自定义 Composables（核心实战）

自定义 Composables 是 Vue 3 逻辑复用的最佳方式，**命名通常以 `use` 开头**（如 `useMouse`、`useFetch`），把相关逻辑封装在一个函数中，最后返回需要暴露给组件的内容。

以下是 3 个最常用的实战示例：

---

### 示例 1：`useMouse` —— 跟踪鼠标位置

封装 “获取鼠标实时位置” 的逻辑，可在任意组件中直接复用。

#### 1. 创建 Composable：`composables/useMouse.js`

```javascript
import { ref, onMounted, onBeforeUnmount } from 'vue'

// 导出函数，命名以 use 开头
export function useMouse() {
  // 定义响应式数据：鼠标 X/Y 坐标
  const x = ref(0)
  const y = ref(0)

  // 定义更新坐标的函数
  const updatePosition = (e) => {
    x.value = e.clientX
    y.value = e.clientY
  }

  // 组件挂载时：绑定鼠标移动事件
  onMounted(() => {
    window.addEventListener('mousemove', updatePosition)
  })

  // 组件卸载前：移除事件监听（避免内存泄漏）
  onBeforeUnmount(() => {
    window.removeEventListener('mousemove', updatePosition)
  })

  // 返回需要暴露给组件的内容
  return { x, y }
}
```

#### 2. 在组件中使用：

```vue
<script setup>
import { useMouse } from './composables/useMouse'

// 直接调用 Composable，获取鼠标坐标
const { x, y } = useMouse()
</script>

<template>
  <div>
    <p>鼠标位置：X: {{ x }}, Y: {{ y }}</p>
  </div>
</template>
```

---

### 示例 2：`useFetch` —— 封装数据请求

封装 “发起 API 请求、处理加载状态、处理错误” 的逻辑，避免每个组件都写重复的请求代码。

#### 1. 创建 Composable：`composables/useFetch.js`

```javascript
import { ref } from 'vue'

export function useFetch(url) {
  const data = ref(null)
  const loading = ref(true)
  const error = ref(null)

  // 定义请求函数
  const execute = async () => {
    loading.value = true
    error.value = null
    try {
      const res = await fetch(url)
      if (!res.ok) throw new Error('请求失败')
      data.value = await res.json()
    } catch (err) {
      error.value = err.message
    } finally {
      loading.value = false
    }
  }

  // 立即执行一次请求
  execute()

  // 返回数据和重新请求的函数
  return { data, loading, error, execute }
}
```

#### 2. 在组件中使用：

vue

```
<script setup>
import { useFetch } from './composables/useFetch'

// 调用 useFetch，传入 API 地址
const { data, loading, error, execute } = useFetch('https://api.example.com/users')
</script>

<template>
  <div>
    <!-- 加载状态 -->
    <div v-if="loading">加载中...</div>
    <!-- 错误状态 -->
    <div v-else-if="error" class="text-red-500">错误：{{ error }}</div>
    <!-- 成功状态 -->
    <div v-else>
      <ul>
        <li v-for="user in data" :key="user.id">{{ user.name }}</li>
      </ul>
      <button @click="execute">重新请求</button>
    </div>
  </div>
</template>
```

---

### 示例 3：`useDebounce` —— 封装防抖逻辑

封装 “防抖” 逻辑（如输入框输入停止后 500ms 再执行搜索），避免频繁触发请求。

#### 1. 创建 Composable：`composables/useDebounce.js`

javascript

运行

```
import { ref, watch } from 'vue'

export function useDebounce(value, delay = 500) {
  const debouncedValue = ref(value.value)
  let timer = null

  // 监听原始值变化
  watch(value, (newVal) => {
    // 清除上一次的定时器
    clearTimeout(timer)
    // 延迟 delay 时间后更新防抖值
    timer = setTimeout(() => {
      debouncedValue.value = newVal
    }, delay)
  })

  return debouncedValue
}
```

#### 2. 在组件中使用：

vue

```
<script setup>
import { ref, watch } from 'vue'
import { useDebounce } from './composables/useDebounce'

const searchText = ref('')
// 对 searchText 进行防抖，延迟 500ms
const debouncedSearchText = useDebounce(searchText, 500)

// 监听防抖后的搜索词，执行搜索
watch(debouncedSearchText, (newVal) => {
  if (newVal) {
    console.log('执行搜索：', newVal)
    // 这里调用搜索 API
  }
})
</script>

<template>
  <div>
    <input v-model="searchText" placeholder="输入搜索内容" />
    <p>防抖后的搜索词：{{ debouncedSearchText }}</p>
  </div>
</template>
```

## 五、最佳实践与注意事项

### 1. 命名规范

- 自定义 Composables 必须以 `use` 开头（如 `useMouse`、`useFetch`），这是 Vue 社区的约定俗成，也让代码更易读。
- 文件名通常与函数名一致（如 `useMouse.js`）。

### 2. 单一职责原则

一个 Composable 只做一件事，不要把多个不相关的逻辑揉在一起（比如不要把 “鼠标位置” 和 “数据请求” 写在同一个 Composable 里），降低维护成本。

### 3. 正确清理副作用

在 `onBeforeUnmount` 钩子中，必须清理 Composable 中绑定的**事件监听、定时器、第三方实例**，否则会造成内存泄漏。

### 4. 返回值的选择

- 如果返回值较多，推荐返回一个**对象**（如 `return { x, y }`），组件中可以按需解构。
- 如果返回值只有一个，也可以直接返回该值（如 `return debouncedValue`）。

### 5. 灵活组合

多个 Composables 可以互相组合使用（比如在 `useFetch` 中使用 `useDebounce`），构建更复杂的逻辑。