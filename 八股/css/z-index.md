`z-index` 是 CSS 中**控制定位元素在三维 z 轴（垂直于屏幕方向）堆叠顺序**的核心属性。它决定了多个重叠元素中 “谁在上、谁在下”—— 数值越大，元素越靠近用户视线，层级越高；数值越小，越远离用户，层级越低。

## 一、基础规则与生效条件

### 1. 生效前提：必须是 “定位元素”

`z-index` **仅对 `position` 非 `static` 的元素生效**，普通静态元素（默认 `position: static`）设置 `z-index` 完全无效。

有效定位类型包括：

- `position: relative`（相对定位）
- `position: absolute`（绝对定位）
- `position: fixed`（固定定位）
- `position: sticky`（粘性定位）

**示例：**

```css
/* 无效：static 元素设 z-index 没用 */
.box1 {
  position: static;
  z-index: 100; 
}

/* 有效：relative 元素可通过 z-index 调整层级 */
.box2 {
  position: relative;
  z-index: 100; 
}
```

### 2. 取值规则

`z-index` 支持以下取值：

- **正整数**：如 `1`、`10`、`999`，数值越大层级越高；
- **负整数**：如 `-1`、`-10`，数值越小层级越低（可能被普通元素覆盖）；
- **`0`**：特殊中间值，与 `auto` 表现相似但有本质区别（见下文 “堆叠上下文”）；
- **`auto`**：默认值，此时元素不形成新的堆叠上下文，层级由 HTML 出现顺序决定（“后来者居上”）。

### 3. 同层级堆叠规则

当多个元素 `z-index` 数值相同（或均为 `auto`）时，遵循 **“后来者居上”** 原则：HTML 结构中后出现的元素，会覆盖先出现的元素。

**示例：**

```html
<div class="box red"></div>
<div class="box blue"></div>
```

```css
.box {
  position: absolute;
  width: 100px;
  height: 100px;
}
.red { background: red; left: 0; top: 0; }
.blue { background: blue; left: 50px; top: 50px; }
```

此时蓝色盒子（后出现）会覆盖红色盒子的重叠部分。

## 二、核心难点：堆叠上下文（Stacking Context）

堆叠上下文是 `z-index` 最容易 “失效” 或 “层级混乱” 的根源，必须深入理解。

### 1. 什么是堆叠上下文？

堆叠上下文是一个**独立的层级 “容器”**：

- 容器内部的元素，`z-index` 仅在该容器内比较高低；
- 容器整体的层级，由其自身在父级堆叠上下文中的 `z-index` 决定；
- 内部元素的 `z-index` 再大，也无法超越外部更高层级的堆叠上下文。

**类比理解：** 堆叠上下文就像 “文件夹”，内部的 “文件”（子元素）只能在文件夹内排序，无法直接跳到其他 “文件夹”（外部堆叠上下文）的上面。

### 2. 哪些情况会形成堆叠上下文？

除了根元素 `<html>`（默认形成），以下情况均会**隐式或显式**创建新的堆叠上下文：

|触发条件|说明|
|---|---|
|`position: absolute/relative` + `z-index` 非 `auto`|显式设置 `z-index` 时触发|
|`position: fixed/sticky`|无论 `z-index` 是否为 `auto`，均触发|
|`flex`/`grid` 容器的子元素 + `z-index` 非 `auto`|子元素需是 flex/grid 项，且 `z-index` 不为 `auto`|
|`opacity < 1`|元素透明度小于 1 时触发（如 `opacity: 0.9`）|
|`transform` 非 `none`|如 `transform: translate(10px)`|
|`filter` 非 `none`|如 `filter: blur(5px)`|
|`perspective` 非 `none`|3D 变换相关属性|
|`isolation: isolate`|显式强制创建堆叠上下文（常用技巧）|

### 3. 堆叠上下文内的层级顺序（从下到上）

在一个堆叠上下文内，元素的层级遵循**固定的 7 层规则**（从最底层到最顶层）：

1. **堆叠上下文的根元素**：背景和边框；
2. **`z-index` 为负的子堆叠上下文**：数值越小越靠下；
3. **普通流块级元素**：非定位的 `div`、`p` 等；
4. **浮动元素**：非定位的 `float` 元素；
5. **普通流行内元素**：非定位的 `span`、`img` 等；
6. **`z-index: auto` 或 `z-index: 0` 的定位元素**：以及 `z-index: 0` 的子堆叠上下文；
7. **`z-index` 为正的子堆叠上下文**：数值越大越靠上。

## 三、高频避坑实战（附代码示例）

### 坑 1：`z-index` 设得很大，却还是被覆盖

**原因：** 元素的父级处于低层级的堆叠上下文，子元素 `z-index` 再大也无法超越外部层级。

**示例：**

```html
<!-- 父容器1：z-index: 1 -->
<div class="parent parent1">
  <div class="child child1">我 z-index: 999</div>
</div>

<!-- 父容器2：z-index: 2 -->
<div class="parent parent2">我 z-index: 2</div>
```

```css
.parent {
  position: relative;
  width: 200px;
  height: 200px;
}
.parent1 { z-index: 1; background: lightgray; }
.parent2 { z-index: 2; background: lightblue; margin-top: -150px; }

.child1 {
  position: absolute;
  z-index: 999; /* 看似很大，但仅在 parent1 内有效 */
  background: red;
  width: 100px;
  height: 100px;
}
```

**结果：** 红色子元素（`z-index: 999`）会被蓝色父容器 2（`z-index: 2`）覆盖，因为 parent1 整体层级低于 parent2。

**解决：** 提高 parent1 的 `z-index`，或调整 HTML 结构让 parent2 位于 parent1 之前。

### 坑 2：`opacity` 导致层级意外升高

**原因：** `opacity < 1` 会隐式创建堆叠上下文，可能让原本层级低的元素 “浮上来”。

**示例：**

```html
<div class="box red"></div>
<div class="box blue"></div>
```

```css
.box {
  position: absolute;
  width: 100px;
  height: 100px;
}
.red { background: red; left: 0; top: 0; opacity: 0.9; } /* 触发堆叠上下文 */
.blue { background: blue; left: 50px; top: 50px; }
```

**结果：** 即使蓝色盒子后出现，红色盒子（因 `opacity` 形成堆叠上下文）也可能覆盖蓝色盒子（具体取决于浏览器解析，但层级规则已被打破）。

**解决：** 避免不必要的 `opacity < 1`，或显式给蓝色盒子设置更高的 `z-index`。

### 坑 3：`transform` 导致 `fixed` 定位失效 + 层级混乱

**原因：** 若 `fixed` 元素的祖先有 `transform` 非 `none`，则 `fixed` 会降级为 `absolute` 定位，且祖先会形成堆叠上下文。

**示例：**

```html
<div class="transform-box">
  <div class="fixed-box">我本该固定在视口</div>
</div>
```

```css
.transform-box {
  transform: translate(0); /* 触发堆叠上下文，且让子元素 fixed 失效 */
  width: 200px;
  height: 200px;
  background: lightgray;
}
.fixed-box {
  position: fixed;
  top: 0;
  left: 0;
  background: red;
  width: 100px;
  height: 100px;
}
```

**结果：** 红色盒子不再固定在视口，而是相对于 `transform-box` 定位。

**解决：** 将 `fixed` 元素移出有 `transform` 的祖先容器。

## 四、实用技巧与最佳实践

### 1. 层级数值管理：避免 “9999” 滥用

建议按功能模块分层级，建立团队规范，例如：

- 基础内容：`0`
- 悬浮卡片：`10`
- 导航栏：`100`
- 下拉菜单：`150`
- 弹窗遮罩：`499`
- 弹窗内容：`500`
- 全局提示：`1000`

**进阶：用 CSS 变量管理**

```css
:root {
  --z-dropdown: 150;
  --z-modal-overlay: 499;
  --z-modal: 500;
}

.dropdown { z-index: var(--z-dropdown); }
.modal-overlay { z-index: var(--z-modal-overlay); }
.modal { z-index: var(--z-modal); }
```

### 2. 显式创建堆叠上下文：`isolation: isolate`

若想让某个元素成为独立的层级容器，但不想改变其 `position` 或 `z-index`，可使用 `isolation: isolate`：

```css
.container {
  isolation: isolate; /* 强制创建堆叠上下文，不影响其他属性 */
}
```

### 3. 调试工具：浏览器开发者工具

Chrome/Firefox 开发者工具可直接查看堆叠上下文：

- 打开 “Elements” 面板，选中元素；
- 查看 “Computed” 或 “Layers” 面板，确认是否形成堆叠上下文，以及层级关系。

## 五、总结

- `z-index` 仅对非 `static` 定位元素生效；
- 堆叠上下文是层级管理的核心，内部 `z-index` 不影响外部；
- 避免滥用高数值，用规范或 CSS 变量管理层级；
- 遇到层级混乱时，先检查是否有隐式堆叠上下文（`opacity`、`transform` 等）。