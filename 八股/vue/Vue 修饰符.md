Vue 修饰符是以 **`.`** 开头的特殊后缀，用于**简化常见的 DOM 操作、事件处理或数据绑定逻辑**，让代码更简洁、更易读。以下是 Vue 3 中最常用的修饰符分类介绍：
### 一、事件修饰符（Event Modifiers）

用于处理 DOM 事件的常见行为（如阻止冒泡、阻止默认行为等），直接加在 `v-on`（或 `@`）事件后面。

| 修饰符        | 作用                                             | 示例                                                               |
| ---------- | ---------------------------------------------- | ---------------------------------------------------------------- |
| `.stop`    | 阻止事件冒泡（事件不再向上层元素传播）。                           | `<button @click.stop="handleClick">点击</button>`                  |
| `.prevent` | 阻止元素的默认行为（如表单提交、链接跳转）。                         | `<form @submit.prevent="handleSubmit">提交</form>`                 |
| `.capture` | 启用事件捕获模式（事件从外层元素向内层元素触发）。                      | `<div @click.capture="handleDivClick">外层</div>`                  |
| `.self`    | 仅当事件在**元素自身**触发时才执行（不包括子元素冒泡上来的事件）。            | `<div @click.self="handleDivClick">外层 <button>内层</button></div>` |
| `.once`    | 事件只触发一次（触发后自动移除监听器）。                           | `<button @click.once="handleOnce">只触发一次</button>`                |
| `.passive` | 告诉浏览器事件监听器不会调用 `preventDefault`（提升滚动性能，移动端常用）。 | `<div @scroll.passive="handleScroll">滚动区域</div>`                 |
**示例组合使用**：

```vue
<!-- 同时阻止冒泡和默认行为 -->
<a href="/link" @click.stop.prevent="handleLink">点击</a>
```

### 二、`v-model` 修饰符

用于增强表单元素的双向绑定行为，直接加在 `v-model` 后面。

|修饰符|作用|示例|
|---|---|---|
|`.lazy`|改为在 `change` 事件（而非 `input` 事件）时更新数据（输入时不更新，失焦 / 回车才更新）。|`<input v-model.lazy="message" placeholder="失焦后更新">`|
|`.number`|自动将输入值转换为**数字类型**（否则输入框默认返回字符串）。|`<input v-model.number="age" type="number" placeholder="输入年龄">`|
|`.trim`|自动去除输入值的**首尾空格**。|`<input v-model.trim="username" placeholder="自动去空格">`|

### 三、按键修饰符（Key Modifiers）

用于监听键盘事件的特定按键，直接加在 `@keyup`、`@keydown` 等键盘事件后面。
#### 1. 常用按键别名

|修饰符|对应按键|
|---|---|
|`.enter`|回车键|
|`.tab`|Tab 键|
|`.delete`|Delete/Backspace 键|
|`.esc`|Esc 键|
|`.space`|空格键|
|`.up`/`.down`/`.left`/`.right`|方向键|
**示例**：

```vue
<!-- 按回车键触发提交 -->
<input @keyup.enter="handleSubmit" placeholder="按回车提交">
```

#### 2. 系统修饰键

需配合其他按键一起使用（按下系统键的同时按下其他键才触发）。

|修饰符|对应按键|
|---|---|
|`.ctrl`|Ctrl 键|
|`.alt`|Alt 键|
|`.shift`|Shift 键|
|`.meta`|Windows 键（Win）或 Command 键（Mac）|
**示例**：

```vue
<!-- 按下 Ctrl + S 触发保存 -->
<input @keydown.ctrl.s="handleSave" placeholder="Ctrl+S 保存">
```

#### 3. `.exact` 修饰符

精确控制触发事件的系统键组合（只有按下指定的系统键才触发，不能多按其他系统键）。

**示例**：

```vue
<!-- 只有按下 Ctrl + 点击才触发（不能同时按 Alt/Shift） -->
<button @click.ctrl.exact="handleCtrlClick">Ctrl+点击</button>
```

### 四、鼠标按钮修饰符

用于监听鼠标的特定按键点击事件。

|修饰符|对应按键|
|---|---|
|`.left`|鼠标左键|
|`.right`|鼠标右键|
|`.middle`|鼠标中键|
**示例**：

```vue
<!-- 只有右键点击才触发 -->
<div @click.right="handleRightClick">右键点击我</div>
```

### 五、其他修饰符

- **`.sync`（Vue 2 常用，Vue 3 用 `v-model:propName` 替代）**：用于父子组件 props 的 “双向绑定”（Vue 3 推荐直接用 `v-model:title` 等写法）。
- **`.camel`**：将 kebab-case 的属性名转换为 camelCase（用于绑定 DOM 属性，如 `viewBox`）。