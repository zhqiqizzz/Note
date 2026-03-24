在 Vue 3 (配合 Vue Router 4) 中，`meta` 字段是路由配置中一个**非常强大且灵活**的自定义属性。它允许你在路由对象上附加任意信息，并在导航守卫、组件内部或布局组件中读取这些信息，从而实现**权限控制、页面标题动态设置、布局切换、缓存控制**等功能。

### 1. 基础配置：如何定义 `meta`

在定义路由数组时，直接在路由对象中添加 `meta` 对象即可。

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import Home from '@/views/Home.vue'
import Admin from '@/views/Admin.vue'
import Login from '@/views/Login.vue'

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home,
    meta: { 
      title: '首页', 
      requiresAuth: false, 
      keepAlive: true 
    }
  },
  {
    path: '/admin',
    name: 'Admin',
    component: Admin,
    meta: { 
      title: '后台管理', 
      requiresAuth: true,      // 需要登录
      roles: ['admin', 'super-admin'], // 需要特定角色
      layout: 'AdminLayout',   // 使用后台布局
      keepAlive: false         // 不缓存
    }
  },
  {
    path: '/login',
    name: 'Login',
    component: Login,
    meta: { 
      title: '登录', 
      requiresAuth: false,
      hideInMenu: true         // 不在侧边栏显示
    }
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

export default router
```

> **注意**：`meta` 里的键值对完全由你定义，Vue Router 不会干涉，你可以放任何数据（字符串、布尔值、数组、对象等）。

### 2. 核心应用场景

#### 场景一：全局前置守卫实现权限控制 (最常用)

利用 `router.beforeEach` 读取 `meta.requiresAuth` 和 `meta.roles`。

```javascript
// router/index.js
import NProgress from 'nprogress' // 假设用了进度条

router.beforeEach((to, from, next) => {
  NProgress.start()

  // 1. 动态设置页面标题
  document.title = to.meta.title ? `${to.meta.title} - 我的系统` : '我的系统'

  // 2. 检查是否需要登录
  const token = localStorage.getItem('token')
  
  if (to.meta.requiresAuth && !token) {
    // 需要登录但没 token，跳回登录页
    next({ name: 'Login', query: { redirect: to.fullPath } })
    NProgress.done()
    return
  }

  // 3. 检查角色权限 (如果 meta 里定义了 roles)
  if (to.meta.roles && token) {
    // 假设用户角色存在 localStorage 或 Pinia 中
    const userRole = localStorage.getItem('role') // 例如 'admin'
    
    if (!to.meta.roles.includes(userRole)) {
      // 角色不匹配，重定向到 403 或首页
      next({ name: 'Home' }) // 或者专门的 403 页面
      NProgress.done()
      return
    }
  }

  // 放行
  next()
})

router.afterEach(() => {
  NProgress.done()
})
```

#### 场景二：在组件内部获取 `meta`

在组件的 `setup` 函数中，可以使用 `useRoute` 钩子访问当前路由信息，包括 `meta`。

```vue
<!-- views/Admin.vue -->
<template>
  <div>
    <h1>{{ route.meta.title }}</h1>
    <p v-if="route.meta.keepAlive">此页面会被缓存</p>
    <button v-if="!route.meta.hideInMenu">显示菜单</button>
  </div>
</template>

<script setup>
import { useRoute } from 'vue-router'

const route = useRoute()

// 可以在逻辑中使用
console.log('当前需要的角色:', route.meta.roles) 
// 输出: ['admin', 'super-admin']
</script>
```

#### 场景三：配合 `<keep-alive>` 实现动态缓存

这是 Vue 3 路由懒加载的高级用法。通过 `meta.keepAlive` 控制哪些页面需要缓存。

**第一步：修改 `App.vue` (或主布局文件)**

```vue
<template>
  <router-view v-slot="{ Component, route }">
    <keep-alive :include="cachedViews">
      <component :is="Component" :key="route.name" v-if="route.meta.keepAlive" />
    </keep-alive>
    
    <!-- 不需要缓存的直接渲染 -->
    <component :is="Component" :key="route.name" v-if="!route.meta.keepAlive" />
  </router-view>
</template>

<script setup>
import { ref, watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const cachedViews = ref([])

// 监听路由变化，动态维护缓存列表
watch(
  () => route.name,
  (newName) => {
    // 如果当前路由标记为 keepAlive 且不在列表中，加入
    if (route.meta.keepAlive && !cachedViews.value.includes(newName)) {
      cachedViews.value.push(newName)
    }
    // 可选：如果不想缓存了，可以从数组移除
  },
  { immediate: true }
)
</script>
```

_注意：`keep-alive` 的 `include` 必须匹配组件的 `name` 选项，所以你的页面组件最好显式定义 `name`。_

#### 场景四：动态布局切换

如果你的系统有“前台布局”和“后台布局”，可以根据 `meta.layout` 动态切换包裹组件。

```vue
<!-- App.vue 简化版 -->
<template>
  <component :is="currentLayout">
    <router-view />
  </component>
</template>

<script setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import DefaultLayout from '@/layouts/DefaultLayout.vue'
import AdminLayout from '@/layouts/AdminLayout.vue'

const route = useRoute()

const currentLayout = computed(() => {
  // 默认布局
  if (!route.meta.layout) return DefaultLayout
  
  // 根据 meta 动态返回布局组件
  if (route.meta.layout === 'AdminLayout') return AdminLayout
  
  return DefaultLayout
})
</script>
```

### 3. 重要注意事项

#### 1. 嵌套路由的 `meta` 合并

如果路由是嵌套的，`to.meta` 会包含**所有匹配路由段**的 `meta` 属性。  
**规则**：子路由的 `meta` 会覆盖父路由的同名属性，未冲突的属性会合并。

```javascript
{
  path: '/dashboard',
  component: DashboardLayout,
  meta: { requiresAuth: true, layout: 'Admin' },
  children: [
    {
      path: 'stats',
      component: Stats,
      meta: { requiresAuth: false } // 覆盖了父级的 requiresAuth
    }
  ]
}
// 访问 /dashboard/stats 时：
// to.meta = { requiresAuth: false, layout: 'Admin' }
```

#### 2. TypeScript 类型支持

如果你使用 TypeScript，直接写 `to.meta.xxx` 可能会报类型错误（因为默认 `meta` 是 `Record<string, any>` 或 `undefined`）。  
建议扩展 `RouteMeta` 接口：

```typescript
// src/types/router.d.ts (或在 router/index.ts 中)
import 'vue-router'

declare module 'vue-router' {
  interface RouteMeta {
    title?: string
    requiresAuth?: boolean
    roles?: string[]
    layout?: string
    keepAlive?: boolean
    hideInMenu?: boolean
  }
}
```

这样在 `to.meta.title` 时就会有智能提示和类型检查了。

#### 3. 响应式问题

`route.meta` 是响应式的。如果你在组件内使用了 `const route = useRoute()`，当路由参数变化（即使是同一路由不同参数）导致重新解析时，`route.meta` 会自动更新。但在 `beforeEach` 守卫中，它是静态快照。

### 总结

|功能|配置示例 (`meta`)|实现位置|
|---|---|---|
|**页面标题**|`{ title: '关于我' }`|`router.beforeEach` -> `document.title`|
|**登录验证**|`{ requiresAuth: true }`|`router.beforeEach` -> 检查 Token|
|**角色权限**|`{ roles: ['admin'] }`|`router.beforeEach` -> 比对用户角色|
|**页面缓存**|`{ keepAlive: true }`|`App.vue` -> `<keep-alive include>`|
|**布局切换**|`{ layout: 'AdminLayout' }`|`App.vue` -> 动态组件 `<component>`|
|**菜单隐藏**|`{ hideInMenu: true }`|侧边栏组件 -> `v-if="!item.meta.hideInMenu"`|

`meta` 是 Vue Router 连接“路由配置”与“业务逻辑”的桥梁，合理使用可以让你的代码结构更清晰，避免硬编码。