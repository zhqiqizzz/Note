Pinia 是 **Vue 3 官方推荐的新一代状态管理库**，作为 Vuex 的 “精神继任者”，它以**极简 API、完美 TypeScript 支持、直观的模块化设计**，实现了更优雅的全局状态共享与跨组件通信。

### 一、核心概念

Pinia 的核心是 **Store（仓库）**，一个 Store 就是一个独立的状态管理单元，包含三个部分：

| 概念          | 作用                                               | 类比组件       |
| ----------- | ------------------------------------------------ | ---------- |
| **State**   | 响应式状态（存储数据）                                      | `data`     |
| **Getters** | 计算属性（基于 State 派生数据，自动缓存）                         | `computed` |
| **Actions** | 方法（修改 State、处理同步 / 异步逻辑，直接修改 State 无需 Mutations） | `methods`  |
### 二、快速上手

#### 1. 安装

```bash
npm install pinia
# 或
yarn add pinia
```

#### 2. 在 Vue 项目中注册

在 `main.js`（或 `main.ts`）中：

```javascript
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia) // 注册 Pinia
app.mount('#app')
```

#### 3. 创建一个 Store

##### Options Store（传统写法）

新建 `src/stores/user.js`（示例：管理用户信息）：

```javascript
import { defineStore } from 'pinia'

// defineStore('仓库唯一ID', { 配置 })
export const useUserStore = defineStore('user', {
  // 1. State：响应式状态（必须用函数返回对象，避免模块间状态污染）
  state: () => ({
    username: '张三',
    age: 25,
    hobbies: ['读书', '游泳']
  }),

  // 2. Getters：计算属性（缓存结果，依赖变化才重新计算）
  getters: {
    welcomeMessage: (state) => `你好，${state.username}！`,
    isAdult: (state) => state.age >= 18,
    // 也可以用 this 访问其他 Getters（注意不能用箭头函数）
    doubleAge() {
      return this.age * 2
    }
  },

  // 3. Actions：方法（支持同步/异步，直接修改 State）
  actions: {
    updateName(newName) {
      this.username = newName // 直接修改 State，无需 Mutations
    },
    addHobby(hobby) {
      this.hobbies.push(hobby)
    },
    // 异步 Action（模拟请求）
    async fetchUserInfo() {
      const res = await new Promise(resolve => {
        setTimeout(() => resolve({ username: '李四', age: 30 }), 1000)
      })
      this.username = res.username
      this.age = res.age
    }
  }
})
```

##### Setup Store（推荐写法：直接写属性和函数）

Setup Store 用 **Composition API** 的方式定义：

- `ref`/`reactive` → 替代 `state`
- `computed` → 替代 `getters`
- 普通函数 → 替代 `actions`
- 最后 `return` 暴露给组件的内容

```javascript
// stores/user.js (Setup Store 风格)
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // --- 1. State：用 ref/reactive 定义响应式数据 ---
  const username = ref('张三')
  const age = ref(25)
  const hobbies = ref(['读书', '游泳'])

  // --- 2. Getters：用 computed 定义计算属性 ---
  const welcomeMessage = computed(() => `你好，${username.value}！`)
  const isAdult = computed(() => age.value >= 18)

  // --- 3. Actions：用普通函数定义方法（直接修改 ref） ---
  const updateName = (newName) => {
    username.value = newName
  }

  const addHobby = (hobby) => {
    hobbies.value.push(hobby)
  }

  const fetchUserInfo = async () => {
    const res = await new Promise(resolve => {
      setTimeout(() => resolve({ username: '李四', age: 30 }), 1000)
    })
    username.value = res.username
    age.value = res.age
  }

  // --- 4. 必须 return 暴露给组件的内容 ---
  return {
    username,
    age,
    hobbies,
    welcomeMessage,
    isAdult,
    updateName,
    addHobby,
    fetchUserInfo
  }
})
```
#### 4. 在组件中使用 Store

```vue
<template>
  <div>
    <h2>用户信息</h2>
    <!-- 读取 State 和 Getters -->
    <p>用户名：{{ userStore.username }}</p>
    <p>年龄：{{ userStore.age }}</p>
    <p>欢迎语：{{ userStore.welcomeMessage }}</p>
    <p>是否成年：{{ userStore.isAdult ? '是' : '否' }}</p>
    <p>爱好：{{ userStore.hobbies.join(', ') }}</p>

    <hr>
    <!-- 调用 Actions -->
    <button @click="userStore.updateName('王五')">修改用户名</button>
    <button @click="userStore.addHobby('编程')">添加爱好</button>
    <button @click="userStore.fetchUserInfo">异步获取用户信息</button>
  </div>
</template>

<script setup>
import { useUserStore } from '@/stores/user'
import { storeToRefs } from 'pinia' // 用于保持响应式解构

const userStore = useUserStore() // 获取 Store 实例

// 【可选】如果想解构 State/Getters 且保持响应式，需用 storeToRefs
// const { username, age, welcomeMessage } = storeToRefs(userStore)
</script>
```

### 三、核心优势

1. **极简 API**：去掉了 Vuex 的 `Mutations`，直接在 `Actions` 中修改 State，代码量减少 40%+。
2. **TypeScript 原生支持**：无需额外配置，自动类型推断，开发体验极佳。
3. **Vue DevTools 深度集成**：可在 DevTools 中查看 State 变化、时间旅行调试、追踪 Actions 调用。
4. **模块化设计**：每个 Store 独立存在，可按需引入，避免 “大而全” 的单一 Store。
5. **无冗余概念**：没有 `Modules`、`Namespacing` 等复杂概念，学习成本极低。

### 四、常用插件：持久化存储

实际项目中，常需将 Store 状态持久化到 `localStorage`/`sessionStorage`（如登录态、用户信息），可使用 `pinia-plugin-persistedstate` 插件：
#### 1. 安装

```bash
npm install pinia-plugin-persistedstate
```

#### 2. 注册插件

在 `main.js` 中：

```javascript
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate) // 注册持久化插件
```

#### 3. 在 Store 中启用持久化

修改 `src/stores/user.js`：

```javascript
export const useUserStore = defineStore('user', {
  state: () => ({ /* ... */ }),
  getters: { /* ... */ },
  actions: { /* ... */ },
  // 启用持久化（默认存储到 localStorage，key 为仓库 ID）
  persist: true
})
```

#### 进阶配置

```javascript
persist: {
  key: 'my-user-store', // 自定义存储的 key
  storage: sessionStorage, // 切换到 sessionStorage
  paths: ['username', 'age'] // 只持久化指定 State（默认全部）
}
```

### 五、使用场景

- **多组件共享状态**：如用户登录态、全局主题、购物车数据。
- **跨层级 / 跨组件通信**：替代复杂的 props/emit 链或事件总线。
- **异步数据管理**：统一管理 API 请求、缓存响应数据。
- **全局配置**：如应用配置、权限列表、字典数据。

### 六、注意事项

- **Store 是单例的**：`useUserStore()` 每次返回的都是同一个实例，确保状态全局唯一。
- **避免直接解构 State**：直接解构会丢失响应式，如需解构请用 `storeToRefs`。
- **Actions 中不要用箭头函数**：否则 `this` 无法指向 Store 实例。