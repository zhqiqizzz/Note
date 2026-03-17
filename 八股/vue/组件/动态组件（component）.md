Vue.js 的 **`<component>`** 是一个**内置动态组件**，通过 `:is` 属性动态绑定不同的组件 / 元素，实现**同一个挂载点灵活切换渲染内容**的效果（类似 “占位符”，根据数据动态填充组件）。
### 一、核心作用

- **动态切换组件**：无需多个 `v-if`/`v-else` 分支，代码更简洁。
- **场景适配**：适合标签页、多步骤流程、权限控制内容等需要灵活切换视图的场景。

### 二、基本用法

通过 `:is` 绑定**组件名字符串**或**组件对象**即可动态渲染。
#### 示例：标签页切换

```vue
<template>
  <div>
    <!-- 切换按钮 -->
    <button @click="currentTab = 'TabHome'">首页</button>
    <button @click="currentTab = 'TabProfile'">个人中心</button>
    <button @click="currentTab = 'TabSettings'">设置</button>

    <!-- 动态组件挂载点 -->
    <component :is="currentTab"></component>
  </div>
</template>

<script setup>
import { ref } from 'vue'
// 引入需要切换的组件
import TabHome from './TabHome.vue'
import TabProfile from './TabProfile.vue'
import TabSettings from './TabSettings.vue'

// 绑定当前显示的组件名
const currentTab = ref('TabHome')
</script>
```

### 三、结合 `<keep-alive>` 缓存组件

动态组件切换时默认会**销毁重建**组件（丢失状态），可通过 `<keep-alive>` 包裹实现**状态保留**（如表单输入、滚动位置）。
#### 示例：缓存动态组件

```vue
<template>
  <div>
    <button @click="currentTab = 'TabForm'">表单</button>
    <button @click="currentTab = 'TabList'">列表</button>

    <!-- 用 keep-alive 包裹，切换时保留状态 -->
    <keep-alive include="TabForm,TabList">
      <component :is="currentTab"></component>
    </keep-alive>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import TabForm from './TabForm.vue'
import TabList from './TabList.vue'
const currentTab = ref('TabForm')
</script>
```

⚠️ 注意：被缓存的组件需设置 `name` 选项（`include`/`exclude` 依赖 `name` 匹配），且会触发 `activated`/`deactivated` 生命周期钩子。

### 四、给动态组件传递 Props 与监听事件

动态组件和普通组件完全一致，支持通过 `v-bind` 传 Props、`v-on` 监听事件。
#### 示例：Props 与事件传递

```vue
<template>
  <div>
    <button @click="currentTab = 'TabA'">组件 A</button>
    <button @click="currentTab = 'TabB'">组件 B</button>

    <!-- 给动态组件传值、监听事件 -->
    <component 
      :is="currentTab" 
      :message="parentMsg" 
      @update="handleUpdate"
    ></component>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import TabA from './TabA.vue'
import TabB from './TabB.vue'

const currentTab = ref('TabA')
const parentMsg = ref('来自父组件的消息')

const handleUpdate = (val) => {
  console.log('子组件触发了 update，值为：', val)
}
</script>
```

### 五、常见使用场景

1. **标签页切换**：如顶部 / 侧边栏导航，切换不同内容区域。
2. **多步骤表单**：如 “第一步→第二步→第三步”，每一步是一个独立组件。
3. **权限控制内容**：根据用户权限动态渲染不同组件（如管理员面板 / 普通用户面板）。
4. **弹窗 / 抽屉内容**：同一个弹窗容器，根据场景显示不同的内容组件。

### 六、注意事项

1. **`:is` 的值类型**：
    
    - 可以是**组件名字符串**（需确保组件已注册）。
    - 也可以是**组件对象**（直接绑定引入的组件变量）。
    
2. **与 `v-if` 的区别**：
    
    - `<component :is>` 适合**多个平级组件切换**（代码更简洁）。
    - `v-if` 适合**简单的条件渲染**（2-3 个分支）。
    
3. **性能优化**：若切换频繁，务必配合 `<keep-alive>` 使用，避免重复渲染。