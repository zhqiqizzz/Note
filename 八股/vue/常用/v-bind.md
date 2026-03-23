`v-bind` 是 Vue 中最核心的指令之一，用于**动态绑定 HTML 元素的属性**（Attribute）或组件的 props，让静态属性变成 “响应式” 的 —— 当绑定的数据变化时，属性值会自动更新。

## 一、基础语法与简写

### 1. 完整语法

```html
<元素 v-bind:属性名="JavaScript表达式"></元素>
```

### 2. 常用简写（推荐）

用 **`:`** 直接替代 `v-bind:`，是开发中最常用的写法：

```html
<元素 :属性名="JavaScript表达式"></元素>
```

## 二、常见应用场景

### 1. 绑定基础属性（`src`/`href`/`title` 等）

最基础的用法，让图片地址、链接地址等随数据动态变化。

```vue
<template>
  <div>
    <!-- 动态绑定图片 src -->
    <img :src="imageUrl" :alt="imageAlt" />

    <!-- 动态绑定链接 href 和 title -->
    <a :href="linkUrl" :title="linkTitle">点击跳转</a>
  </div>
</template>

<script setup>
import { ref } from 'vue'
const imageUrl = ref('https://example.com/logo.png')
const imageAlt = ref('网站Logo')
const linkUrl = ref('https://vuejs.org')
const linkTitle = ref('Vue官方文档')
</script>
```

### 2. 绑定布尔属性（`disabled`/`checked`/`hidden` 等）

对于 `disabled`、`checked` 这类布尔属性，当绑定的表达式为**真值**时，属性会被添加到元素上；为**假值**时，属性会被移除。

```vue
<template>
  <div>
    <!-- 按钮是否禁用：isDisabled 为 true 时禁用 -->
    <button :disabled="isDisabled">提交</button>

    <!-- 复选框是否选中：isChecked 为 true 时选中 -->
    <input type="checkbox" :checked="isChecked" />
  </div>
</template>

<script setup>
import { ref } from 'vue'
const isDisabled = ref(true)
const isChecked = ref(false)
</script>
```

### 3. 动态绑定 `class`（最常用）

`v-bind:class` 支持**对象语法**和**数组语法**，灵活控制元素的类名。
#### （1）对象语法

类名是否存在，取决于对象的值是否为真值：

```vue
<template>
  <div 
    :class="{ 
      active: isActive, 
      'text-danger': hasError, 
      'font-bold': isBold 
    }"
  >
    动态类名
  </div>
</template>

<script setup>
import { ref } from 'vue'
const isActive = ref(true)
const hasError = ref(false)
const isBold = ref(true)
</script>
```

最终渲染的类名：`active font-bold`（`hasError` 为 `false`，所以 `text-danger` 不出现）。

#### （2）数组语法

按顺序应用数组中的类名，也可以在数组中嵌套对象：

```vue
<template>
  <div :class="[baseClass, isActive ? 'active' : '', { 'text-danger': hasError }]">
    数组语法类名
  </div>
</template>

<script setup>
import { ref } from 'vue'
const baseClass = ref('box')
const isActive = ref(true)
const hasError = ref(false)
</script>
```

### 4. 动态绑定 `style`（内联样式）

`v-bind:style` 同样支持**对象语法**和**数组语法**，注意 CSS 属性名要用驼峰式（如 `fontSize`）或短横线式（加引号，如 `'font-size'`）。
#### （1）对象语法

```vue
<template>
  <div :style="{ color: textColor, fontSize: fontSize + 'px', backgroundColor: bgColor }">
    动态内联样式
  </div>
</template>

<script setup>
import { ref } from 'vue'
const textColor = ref('red')
const fontSize = ref(16)
const bgColor = ref('#f0f0f0')
</script>
```
#### （2）数组语法

将多个样式对象合并应用：

```vue
<template>
  <div :style="[baseStyle, extraStyle]">
    数组语法内联样式
  </div>
</template>

<script setup>
import { reactive } from 'vue'
const baseStyle = reactive({ color: 'blue', fontSize: '16px' })
const extraStyle = reactive({ fontWeight: 'bold', backgroundColor: '#e0e0e0' })
</script>
```

## 三、进阶用法

### 1. 动态属性名

用方括号 `[]` 包裹属性名，让属性名本身也能动态变化：

```vue
<template>
  <div>
    <!-- 动态属性名：attrName 的值决定了绑定的是哪个属性 -->
    <button :[attrName]="attrValue">按钮</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
// 可以动态改变为 'disabled'、'title' 等
const attrName = ref('title')
const attrValue = ref('这是一个提示')
</script>
```

### 2. 绑定对象（批量传递属性 / Props）

如果需要给元素或组件传递多个属性，可以直接绑定一个对象，对象的键会作为属性名，值作为属性值：

```vue
<template>
  <div>
    <!-- 批量绑定 HTML 属性 -->
    <div v-bind="attrs">批量绑定属性</div>

    <!-- 批量传递组件 Props -->
    <MyComponent v-bind="componentProps" />
  </div>
</template>

<script setup>
import { reactive } from 'vue'
import MyComponent from './MyComponent.vue'

// 批量 HTML 属性
const attrs = reactive({
  id: 'my-div',
  class: 'box',
  title: '这是一个div'
})

// 批量组件 Props
const componentProps = reactive({
  name: '张三',
  age: 25,
  gender: '男'
})
</script>
```

### 3. Vue 3 专属：CSS 中使用 `v-bind`

Vue 3 支持在 `<style>` 标签中直接用 `v-bind` 绑定响应式数据，让 CSS 也能随数据变化：

```vue
<template>
  <div class="box">CSS 中使用 v-bind</div>
  <button @click="color = 'blue'">改变颜色</button>
</template>

<script setup>
import { ref } from 'vue'
const color = ref('red')
</script>

<style scoped>
.box {
  /* 直接绑定响应式数据 color */
  color: v-bind(color);
  font-size: 20px;
}
</style>
```

## 四、关键注意事项

1. **绑定的是 JavaScript 表达式**：`v-bind` 后面的引号内是一个完整的 JS 表达式，可以写简单运算、三元运算符、函数调用等（但不要写复杂逻辑，推荐用 `computed`）。
    
```html
    <!-- 简单运算 -->
    <p :class="isActive ? 'active' : ''">文本</p>
    <!-- 函数调用 -->
    <p :title="getTitle()">鼠标悬停</p>
```
    
2. **布尔属性的真假值**：`null`、`undefined`、`false` 会让布尔属性（如 `disabled`）被移除；`0`、`''`（空字符串）会被当作真值，属性会保留。