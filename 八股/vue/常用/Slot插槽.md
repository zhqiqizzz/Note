Vue 的 **插槽（Slot）** 是一种**内容分发机制**，允许父组件向子组件 “传递” 自定义内容，让子组件更灵活、更具复用性（类似 HTML 元素的 “开口”，父组件可以往里面填内容）。

## 一、核心概念

- **子组件**：用 `<slot>` 标签作为**占位符**，标记父组件内容的插入位置。
- **父组件**：在子组件标签内部编写内容，这些内容会替换子组件的 `<slot>`。

## 二、三种插槽类型

> 默认插槽、具名插槽、作用域插槽
### 1. 默认插槽（Default Slot）

最简单的插槽，子组件只有一个 “开口”，父组件直接传递内容。
#### 子组件（定义插槽）：

```vue
<!-- MyButton.vue -->
<template>
  <button class="my-button">
    <!-- slot 是占位符，父组件的内容会替换这里 -->
    <!-- 如果父组件没传内容，显示“后备内容”：默认按钮 -->
    <slot>默认按钮</slot>
  </button>
</template>
```

#### 父组件（使用插槽）：

```vue
<template>
  <div>
    <!-- 不传内容：显示子组件的后备内容“默认按钮” -->
    <MyButton />

    <!-- 传内容：替换 slot -->
    <MyButton>点击我</MyButton>
    <MyButton><strong>加粗按钮</strong></MyButton>
  </div>
</template>

<script setup>
import MyButton from './MyButton.vue'
</script>
```

### 2. 具名插槽（Named Slot）

当子组件需要 **多个 “开口”** 时，给 `<slot>` 加 `name` 属性区分，父组件用 `v-slot:name`（或简写 `#name`）指定内容插入的位置。
#### 子组件（定义多个具名插槽）：

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <!-- 具名插槽：name="header" -->
    <div class="card-header">
      <slot name="header">默认头部</slot>
    </div>

    <!-- 默认插槽：name="default"（可省略 name） -->
    <div class="card-body">
      <slot>默认内容</slot>
    </div>

    <!-- 具名插槽：name="footer" -->
    <div class="card-footer">
      <slot name="footer">默认底部</slot>
    </div>
  </div>
</template>
```

#### 父组件（使用具名插槽）：

用 `<template v-slot:name>` 包裹内容，或简写为 `<template #name>`。

```vue
<template>
  <Card>
    <!-- 插入 header 插槽 -->
    <template #header>
      <h3>卡片标题</h3>
    </template>

    <!-- 插入默认插槽（可省略 #default，直接写内容） -->
    <p>这是卡片的主体内容</p>

    <!-- 插入 footer 插槽 -->
    <template #footer>
      <button>确定</button>
    </template>
  </Card>
</template>

<script setup>
import Card from './Card.vue'
</script>
```

### 3. 作用域插槽（Scoped Slot）

**核心功能**：子组件可以向父组件**传递数据**，父组件接收数据后自定义渲染内容（类似 “子组件给父组件传参”）。

#### 子组件（向插槽传递数据）：

用 `:属性名="数据"` 的方式向 `<slot>` 传值。

```vue
<!-- List.vue -->
<template>
  <ul>
    <li v-for="item in list" :key="item.id">
      <!-- 向插槽传递 item 和 index -->
      <slot :item="item" :index="index">
        <!-- 后备内容：如果父组件没传，默认显示 item.name -->
        {{ item.name }}
      </slot>
    </li>
  </ul>
</template>

<script setup>
const list = [
  { id: 1, name: '苹果', price: 5 },
  { id: 2, name: '香蕉', price: 3 }
]
</script>
```

#### 父组件（接收并使用插槽数据）：

用 `v-slot="slotProps"` 接收数据（`slotProps` 是一个对象，包含子组件传递的所有属性），也可以**解构**直接拿到需要的属性。

```vue
<template>
  <div>
    <!-- 方式1：用 slotProps 接收所有数据 -->
    <List v-slot="slotProps">
      <span>{{ slotProps.index + 1 }}. {{ slotProps.item.name }} - ￥{{ slotProps.item.price }}</span>
    </List>

    <hr>

    <!-- 方式2：解构接收（更简洁，推荐） -->
    <List v-slot="{ item, index }">
      <span :style="{ color: item.price > 4 ? 'red' : 'green' }">
        {{ index + 1 }}. {{ item.name }} - ￥{{ item.price }}
      </span>
    </List>

    <hr>

    <!-- 具名插槽 + 作用域插槽：用 #name="slotProps" -->
    <!-- 假设 List 有具名插槽 header，也可以传数据 -->
    <!-- <List #header="{ title }">{{ title }}</List> -->
  </div>
</template>

<script setup>
import List from './List.vue'
</script>
```

## 三、使用场景

1. **布局组件**：如 `Card`、`Modal`、`Layout`（Header/Main/Footer），让父组件自定义各部分内容。
2. **列表组件**：如 `List`、`Table`，让父组件自定义每一项的渲染方式（作用域插槽最常用的场景）。
3. **通用组件**：如 `Button`、`Input`，让父组件自定义图标、文字等内容。

## 四、注意事项

- **`v-slot` 的位置**：只能用在 `<template>` 或**组件标签**上（当只有默认插槽时，可直接用在组件标签上）。
- **具名插槽的名称**：父组件的 `v-slot:name` 必须与子组件的 `<slot name="name">` 完全对应。
- **作用域插槽的数据**：只能在 `<template v-slot>` 内部使用，外部无法访问。