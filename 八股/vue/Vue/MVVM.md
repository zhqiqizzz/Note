**MVVM**（Model-View-ViewModel）是一种**软件架构设计模式**，核心思想是通过**数据绑定**实现**视图（View）与逻辑（ViewModel）的解耦**，让开发者专注于业务逻辑而非 DOM 操作。Vue.js、Angular 等前端框架的设计理念都深度契合 MVVM。

### 一、核心组成

MVVM 将应用分为三层，各司其职：

| 层级            | 作用                                         |
| ------------- | ------------------------------------------ |
| **Model**     | 数据层：负责数据的存储、获取（如 API 请求、本地存储），不涉及 UI。      |
| **View**      | 视图层：负责 UI 展示（用户看到的界面），通过模板语法绑定数据。          |
| **ViewModel** | 视图模型层：**连接 Model 和 View 的桥梁**，处理业务逻辑、数据转换。 |

### 二、各层详解

#### 1. Model（数据层）

- **职责**：管理纯数据（如用户信息、商品列表），不关心数据如何展示。
- **示例**：

```javascript
    // Model：从 API 获取用户数据
    const UserModel = {
      async fetchUser() {
        const res = await fetch('/api/user')
        return res.json() // 返回纯数据：{ name: '张三', age: 25 }
      }
    }
```
#### 2. View（视图层）

- **职责**：仅负责 UI 渲染，通过模板语法绑定 ViewModel 的数据和事件，**不写复杂逻辑**。
- **示例（Vue 模板）**：
```vue
    <!-- View：UI 模板，通过 {{ }}、v-bind、v-on 绑定数据和事件 -->
    <template>
      <div class="user-card">
        <p>姓名：{{ user.name }}</p>
        <p>年龄：{{ user.age }}</p>
        <button @click="updateName">修改姓名</button>
      </div>
    </template>
```
#### 3. ViewModel（视图模型层，核心）

- **职责**：
    
    1. 从 Model 获取数据，转换为 View 可展示的格式。
    2. 处理 View 的交互（如点击事件），更新 Model 或自身数据。
    3. 通过**数据绑定**自动同步 View 和 ViewModel 的状态（无需手动操作 DOM）。
    
- **示例（Vue 组件的 Script 部分）**：

```vue
    <script setup>
    import { ref, onMounted } from 'vue'
    import UserModel from './model'
    
    // ViewModel：响应式数据 + 业务逻辑
    const user = ref({ name: '', age: 0 })
    
    // 从 Model 获取数据
    const loadUser = async () => {
      user.value = await UserModel.fetchUser()
    }
    
    // 处理 View 的点击事件
    const updateName = () => {
      user.value.name = '李四' // 直接修改数据，View 自动更新
    }
    
    onMounted(() => loadUser())
    </script>
```
### 三、核心机制：数据绑定

MVVM 的灵魂是**双向数据绑定**（Two-way Data Binding）：

> 那就又来到了 [双向数据绑定原理](../页面渲染/双向数据绑定原理.md) 

- **View → ViewModel**：View 的输入变化（如表单输入）自动更新 ViewModel 的数据。
- **ViewModel → View**：ViewModel 的数据变化自动更新 View 的 UI。

**示例（Vue 的 v-model）**：

```vue
<template>
  <!-- View：输入框内容变化 → ViewModel 的 message 自动更新；
       ViewModel 的 message 变化 → 输入框内容自动更新 -->
  <input v-model="message" placeholder="输入内容">
  <p>你输入的是：{{ message }}</p>
</template>

<script setup>
import { ref } from 'vue'
const message = ref('') // ViewModel 的响应式数据
</script>
```

### 四、MVVM 的优势

1. **解耦**：View 和 ViewModel 分离，修改 UI 不影响业务逻辑，修改逻辑不影响 UI。
2. **可复用**：一个 ViewModel 可以复用到多个 View（如 “用户信息卡片” 和 “用户信息弹窗” 共用同一个 ViewModel）。
3. **开发效率高**：数据绑定自动同步 View 和 ViewModel，无需手动操作 DOM。
4. **易于测试**：ViewModel 不依赖 View，可以单独编写单元测试（无需浏览器环境）。

### 五、MVVM vs MVC（对比理解）

MVVM 是从 MVC（Model-View-Controller）演变而来，核心区别在于**Controller 与 ViewModel 的定位**：

|特性|MVC|MVVM|
|---|---|---|
|连接层|Controller（处理用户输入，手动更新 View）|ViewModel（通过数据绑定自动同步）|
|View 与 Model|可能直接交互|完全解耦，通过 ViewModel 间接交互|
|耦合度|较高（Controller 依赖 View）|较低（ViewModel 不依赖 View）|

### 六、适用场景

- **复杂前端应用**：如后台管理系统、电商平台（UI 频繁更新、逻辑复杂）。
- **需多人协作的项目**：View 和 ViewModel 分离，设计师和开发者可并行工作。

### 一句话总结

MVVM 通过 **Model（数据）→ ViewModel（逻辑 / 桥梁）→ View（UI）** 的分层，配合**数据绑定**，实现了 “数据驱动视图”，让开发者从繁琐的 DOM 操作中解放出来。