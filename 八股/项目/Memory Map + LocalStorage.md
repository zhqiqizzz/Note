这是一种 **「速度优先、持久化兜底」** 的双层缓存方案，核心是利用两者的优势互补：

- **Memory Map**：内存缓存，读写速度极快（纳秒级），但页面刷新 / 关闭后数据丢失，适合做「热缓存」（高频访问的临时数据）；
- **LocalStorage**：磁盘持久化缓存，刷新 / 重启浏览器后数据仍在，但容量有限（约 5MB）、读写速度远慢于内存，适合做「冷缓存」（不常变的持久化数据）。

两者结合，既能享受内存的极致速度，又能通过 LocalStorage 实现跨会话复用，是前端高频数据缓存的经典方案。

## 一、核心设计思路

### 1. 分层定位

| 层级     | 存储介质         | 核心作用          | 优势                           | 劣势                           |
| :----- | :----------- | :------------ | :--------------------------- | :--------------------------- |
| L1 热缓存 | Memory Map   | 优先响应请求，提供极致速度 | 读写极快、无容量限制（受限于设备内存）、支持任意数据类型 | 页面刷新后丢失、非持久化                 |
| L2 冷缓存 | LocalStorage | 持久化兜底，跨会话复用数据 | 持久化存储、跨标签页 / 会话可用            | 容量有限、仅支持字符串、读写较慢、同步操作可能阻塞主线程 |

#### Memory（内存存储）：变量 / 对象 / Map
指用 JavaScript 变量直接存储在**浏览器内存**里的数据，页面刷新、关闭标签页就会**彻底丢失**。

最常用的两种形式：
##### （1）普通对象 / 变量

```javascript
// 普通变量存储
let currentUser = null
let token = ''

// 普通对象存储
const state = {
  userInfo: null,
  messageList: [],
  unreadCount: 0
}
```

##### （2）`Map` 数据结构（更推荐用于键值对存储）

`Map` 是 ES6 新增的键值对集合，比普通对象更灵活：

- 键可以是任意类型（对象、函数、数字等，不限于字符串）；
- 有直接的 `size` 属性获取键值对数量；
- 插入 / 查找 / 删除性能更高，适合频繁操作。

```javascript
// Map 作为内存存储
const memoryCache = new Map()

// 存
memoryCache.set('user_1001', { id: 1001, name: '张三' })
memoryCache.set('currentToken', 'xxxxxx')

// 取
const user = memoryCache.get('user_1001')

// 删
memoryCache.delete('user_1001')

// 清空
memoryCache.clear()
```
#### 2. LocalStorage：持久化本地存储

HTML5 提供的**浏览器本地持久化存储 API**，数据会永久保存在浏览器中（除非用户手动清除缓存、隐私模式或代码主动删除），大小限制约 **5MB**，仅支持存储**字符串**。
##### 核心 API

```javascript
// 1. 存（自动把非字符串转成字符串，建议手动 JSON.stringify）
localStorage.setItem('token', 'xxxxxx')
localStorage.setItem('userInfo', JSON.stringify({ id: 1001, name: '张三' }))

// 2. 取（取出来的是字符串，对象需要 JSON.parse）
const token = localStorage.getItem('token')
const userInfo = JSON.parse(localStorage.getItem('userInfo') || '{}')

// 3. 删
localStorage.removeItem('token')

// 4. 清空所有
localStorage.clear()
```

### 2. 核心原则

1. **优先读内存**：请求数据时，先查 Memory Map，命中则直接返回（最快路径）；
2. **内存未命中查 LocalStorage**：Memory Map 没有时，查 LocalStorage，命中则将数据同步到 Memory Map（为下次请求提速），再返回；
3. **都未命中则请求服务器**：拿到数据后，**同时写入 Memory Map 和 LocalStorage**，既保证当前页面的访问速度，又为下次访问做持久化兜底；
4. **过期淘汰机制**：给缓存项加过期时间，避免数据过时；LocalStorage 满时，用 LRU（最近最少使用）算法淘汰旧数据。

## 二、完整执行流程

我们以「获取用户配置」为例，完整走一遍混合缓存的流程：

1. **发起请求**：调用 `cache.get('userConfig')`；
2. **查 L1 内存缓存**：
    
    - ✅ 命中：直接返回配置数据，流程结束（耗时 < 1ms）；
    - ❌ 未命中：进入下一步；
    
3. **查 L2 持久化缓存**：
    
    - ✅ 命中且未过期：将数据从 LocalStorage 取出，JSON.parse 后存入 Memory Map，返回数据（耗时约 10-50ms）；
    - ❌ 未命中 / 已过期：进入下一步；
    
4. **请求服务器**：发起 API 请求获取用户配置；
5. **回写缓存**：
    
    - 将配置数据存入 Memory Map；
    - 将配置数据 + 过期时间戳，JSON.stringify 后存入 LocalStorage；
    
6. **返回数据**：将服务器返回的配置数据返回给调用方。

这是核心 ——**Memory 负责「当前页面的临时状态和响应式数据」，LocalStorage 负责「持久化保存和页面刷新后的恢复」**。

```plaintext
1. 页面加载 → 从 LocalStorage 读取持久化数据 → 存入 Memory（Vuex/Pinia/变量/Map）
2. 页面运行 → 所有读写操作都在 Memory 中进行（响应式、速度快）
3. 关键数据变更 → 同步保存一份到 LocalStorage（持久化）
4. 页面刷新 → 重复步骤1，从 LocalStorage 恢复数据到 Memory
```

## 三、关键细节实现

### 1. 缓存项结构

为了支持过期时间和 LRU 淘汰，每个缓存项需要包含以下字段：

```javascript
{
  value: any,          // 实际缓存的数据
  expireAt: number,    // 过期时间戳（毫秒）
  lastAccessAt: number // 最后访问时间戳（用于LRU淘汰）
}
```

### 2. 过期时间处理

- 写入缓存时，根据配置的 `ttl`（生存时间，毫秒）计算 `expireAt = Date.now() + ttl`；
- 读取缓存时，判断 `Date.now() > expireAt`，若成立则删除该缓存项，视为未命中；
- 支持 `ttl: 0` 或 `ttl: Infinity` 表示永不过期（需谨慎使用）。

### 3. LRU 淘汰策略（LocalStorage 容量兜底）

LocalStorage 容量有限（约 5MB），写入时若报错（`QuotaExceededError`），需触发淘汰：

1. 遍历 LocalStorage 中所有缓存项，按 `lastAccessAt` 排序；
2. 删除最久未访问的 20% 缓存项；
3. 重新尝试写入，若仍失败则继续淘汰，直到写入成功或清空所有缓存。

### 4. 数据序列化与反序列化

- LocalStorage 仅支持字符串，写入时需用 `JSON.stringify()` 序列化；
- 读取时需用 `JSON.parse()` 反序列化；
- 注意：无法序列化函数、正则、Symbol、BigInt 等特殊类型，若需缓存此类数据，需自定义序列化逻辑（或仅用 Memory Map 缓存）。

## 四、使用示例

以「用户登录态 + 未读消息数」为例，用 Vue 3 + Pinia（Memory）+ LocalStorage 实现：
### 1. Pinia Store（Memory 层，响应式）

```javascript
// stores/user.js
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({
    // Memory 中的响应式数据
    token: '',
    userInfo: null,
    unreadCount: 0
  }),

  actions: {
    // 1. 初始化：从 LocalStorage 恢复数据到 Memory
    initFromStorage() {
      const token = localStorage.getItem('token')
      const userInfo = JSON.parse(localStorage.getItem('userInfo') || 'null')
      const unreadCount = parseInt(localStorage.getItem('unreadCount') || '0')
      
      if (token) {
        this.token = token
        this.userInfo = userInfo
        this.unreadCount = unreadCount
      }
    },

    // 2. 登录：同时更新 Memory 和 LocalStorage
    async login(loginData) {
      // 调用登录接口...
      const res = await loginApi(loginData)
      
      // 更新 Memory（响应式，页面立刻更新）
      this.token = res.data.token
      this.userInfo = res.data.userInfo
      this.unreadCount = res.data.unreadCount
      
      // 同步到 LocalStorage（持久化，刷新不丢）
      localStorage.setItem('token', this.token)
      localStorage.setItem('userInfo', JSON.stringify(this.userInfo))
      localStorage.setItem('unreadCount', this.unreadCount.toString())
    },

    // 3. 更新未读消息数：同时更新 Memory 和 LocalStorage
    updateUnreadCount(count) {
      this.unreadCount = count
      localStorage.setItem('unreadCount', count.toString())
    },

    // 4. 退出登录：同时清空 Memory 和 LocalStorage
    logout() {
      this.token = ''
      this.userInfo = null
      this.unreadCount = 0
      localStorage.removeItem('token')
      localStorage.removeItem('userInfo')
      localStorage.removeItem('unreadCount')
    }
  }
})
```
### 2. 页面入口初始化

```javascript
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import { createPinia } from 'pinia'
import { useUserStore } from './stores/user'

const app = createApp(App)
const pinia = createPinia()
app.use(pinia)

// 页面加载时，从 LocalStorage 恢复数据到 Memory
const userStore = useUserStore()
userStore.initFromStorage()

app.mount('#app')
```

### 3. 业务组件中使用

```vue
<template>
  <div>
    <p>当前用户：{{ userStore.userInfo?.name }}</p>
    <p>未读消息：{{ userStore.unreadCount }}</p>
    <button @click="userStore.logout">退出登录</button>
  </div>
</template>

<script setup>
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()
</script>
```

## 五、补充：Map 作为 Memory 缓存的场景

如果你的招聘软件需要**缓存求职者 / HR 的简历信息**（避免重复请求接口），用 `Map` 做内存缓存非常合适：

```javascript
// utils/cache.js
// 用 Map 做内存缓存，存储简历详情
const resumeCache = new Map()
// 缓存过期时间（比如5分钟）
const CACHE_EXPIRE_TIME = 5 * 60 * 1000

/**
 * 获取简历详情：优先从 Map 缓存取，没有再请求接口
 */
export async function getResumeDetail(resumeId) {
  // 1. 检查缓存是否存在且未过期
  const cached = resumeCache.get(resumeId)
  if (cached && Date.now() - cached.timestamp < CACHE_EXPIRE_TIME) {
    return cached.data // 直接返回缓存，速度极快
  }

  // 2. 缓存没有或已过期，请求接口
  const res = await getResumeApi(resumeId)
  
  // 3. 存入 Map 缓存
  resumeCache.set(resumeId, {
    data: res.data,
    timestamp: Date.now()
  })

  return res.data
}

/**
 * 清除指定简历缓存（比如简历更新后）
 */
export function clearResumeCache(resumeId) {
  resumeCache.delete(resumeId)
}
```

## 六、常见误区与避坑指南

### 1. 不要把所有数据都存 LocalStorage

- **错误**：把聊天消息列表、临时筛选条件都存 LocalStorage；
- **正确**：只存**必须持久化的数据**（登录态、用户偏好、未读数量），其他数据放 Memory。

### 2. LocalStorage 存对象必须 JSON 序列化

```javascript
// ❌ 错误：直接存对象，会变成 "[object Object]"
localStorage.setItem('userInfo', { id: 1001 })

// ✅ 正确：JSON.stringify 序列化
localStorage.setItem('userInfo', JSON.stringify({ id: 1001 }))
// 取的时候 JSON.parse
const userInfo = JSON.parse(localStorage.getItem('userInfo') || '{}')
```

### 3. 不要频繁读写 LocalStorage

LocalStorage 是磁盘 IO，比内存慢，频繁读写会影响性能：

- **正确做法**：平时读写都在 Memory 中，只在关键数据变更时（比如登录、退出、未读数变化）同步一次到 LocalStorage。

### 4. 敏感数据不要存 LocalStorage

密码、身份证号、银行卡号等敏感数据，绝对不要存 LocalStorage，容易被 XSS 攻击窃取。

## 最终总结

|方案|核心作用|你的招聘软件适用场景|
|---|---|---|
|**Memory（变量 / Pinia/Map）**|临时状态、响应式数据、内存缓存|当前登录用户信息、未读消息数、聊天消息列表、简历缓存|
|**LocalStorage**|持久化保存、页面刷新恢复|登录 Token、用户偏好设置、未读消息数（备份）|

**最佳实践**：Memory 为主（响应式、速度快），LocalStorage 为辅（持久化、备份），两者配合使用，兼顾体验和数据安全。