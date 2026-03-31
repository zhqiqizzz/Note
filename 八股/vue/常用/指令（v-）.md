好的，我们重新系统梳理一下 Vue.js 中以 `v-` 开头的**指令（Directives）**—— 它们是 Vue 模板语法的核心，用于直接响应式地操作 DOM。

### 一、核心常用指令

#### 1. `v-bind`：动态属性绑定

- **功能**：将 HTML 元素的属性（如 `src`、`class`、`style`）与 Vue 数据动态绑定。
- **简写**：`:`（冒号）。
- **示例**：
   
```html
    <!-- 绑定图片地址 -->
    <img :src="imgUrl" alt="动态图">
    
    <!-- 绑定 class（对象/数组语法） -->
    <div :class="{ active: isActive, 'text-red': hasError }"></div>
```

#### 2. `v-model`：双向数据绑定

- **功能**：在表单元素（`input`、`textarea`、`select`）与 Vue 数据间建立双向同步（数据变→视图变，视图变→数据变）。
- **示例**：
    
```html
    <input v-model="username" placeholder="请输入用户名">
    <p>你输入的是：{{ username }}</p>
```

#### 3. `v-if` / `v-else-if` / `v-else`：条件渲染

- **功能**：根据表达式真假，**动态插入 / 销毁** DOM 元素（性能开销较大，适合不频繁切换的场景）。
- **示例**：
    
```html
    <p v-if="score >= 90">优秀</p>
    <p v-else-if="score >= 60">及格</p>
    <p v-else>不及格</p>
```

#### 4. `v-show`：条件显示

- **功能**：通过切换 CSS `display` 属性来**显示 / 隐藏**元素（始终保留在 DOM 中，适合频繁切换）。
- **示例**：
    
```html
    <p v-show="isVisible">这段文字会切换显示</p>
```

#### 5. `v-for`：列表渲染

- **功能**：基于数组 / 对象循环渲染元素，必须配合 `:key` 唯一标识（优化渲染性能）。
- **示例**：
    
```html
    <ul>
      <li v-for="(item, index) in list" :key="item.id">
        {{ index + 1 }}. {{ item.name }}
      </li>
    </ul>
```

#### 6. `v-on`：事件监听

- **功能**：监听 DOM 事件（如 `click`、`input`）并触发对应方法。
- **简写**：`@`（艾特符号）。
- **示例**：
    
```html
    <button @click="count++">点击次数：{{ count }}</button>
    <button @click="handleSubmit">提交</button>
```

### 二、其他实用指令

- **`v-text`**：更新元素的 `textContent`（等价于 `{{ }}`，但会覆盖整个内容）。
- **`v-html`**：更新元素的 `innerHTML`（⚠️ 仅用于可信内容，防止 XSS 攻击）。
- **`v-pre`**：跳过该元素及其子元素的编译（直接显示原始 `{{ }}` 语法）。
- **`v-cloak`**：解决插值表达式闪烁问题（配合 CSS `[v-cloak] { display: none; }` 使用）。
- **`v-once`**：仅渲染一次（后续数据变化不更新，用于性能优化）。