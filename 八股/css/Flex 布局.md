Flex 布局（Flexible Box，弹性盒子）是 CSS3 引入的**一维布局模型**，专门用于解决传统布局（如 `float`、`position`）中垂直居中、等高列、空间分配等痛点，是目前前端最主流的布局方式之一。

## 一、核心概念

Flex 布局由 **Flex 容器（Flex Container）** 和 **Flex 项目（Flex Items）** 组成：

- **Flex 容器**：设置了 `display: flex` 或 `display: inline-flex` 的元素，是 Flex 项目的父元素。
- **Flex 项目**：Flex 容器的**直接子元素**（孙子元素不受影响）。

### 关键：主轴与交叉轴

Flex 布局基于**主轴（Main Axis）** 和 **交叉轴（Cross Axis）**：

- **主轴**：项目排列的主要方向（由 `flex-direction` 决定），有起点（main start）和终点（main end）。
- **交叉轴**：垂直于主轴的方向，有起点（cross start）和终点（cross end）。

例如：

- 若 `flex-direction: row`（默认），主轴为**水平方向**，交叉轴为**垂直方向**。
- 若 `flex-direction: column`，主轴为**垂直方向**，交叉轴为**水平方向**。

## 二、Flex 容器的属性（6 个核心属性）

### 1. `flex-direction`：决定主轴方向

定义项目在主轴上的排列方向。

|属性值|说明|
|---|---|
|`row`（默认）|主轴为水平方向，起点在左，项目从左到右排列。|
|`row-reverse`|主轴为水平方向，起点在右，项目从右到左排列。|
|`column`|主轴为垂直方向，起点在上，项目从上到下排列。|
|`column-reverse`|主轴为垂直方向，起点在下，项目从下到上排列。|

**示例**：

```css
.container {
  display: flex;
  flex-direction: row; /* 项目从左到右排列 */
}
```

### 2. `flex-wrap`：决定项目是否换行

定义当项目在主轴上排不下时，是否换行。

|属性值|说明|
|---|---|
|`nowrap`（默认）|不换行，项目会被压缩（即使设置了宽度）。|
|`wrap`|换行，第一行在上方。|
|`wrap-reverse`|换行，第一行在下方。|
**示例**：

```css
.container {
  display: flex;
  flex-wrap: wrap; /* 排不下时换行 */
}
```

### 3. `flex-flow`：`flex-direction` + `flex-wrap` 的简写

合并设置主轴方向和换行，默认值为 `row nowrap`。

**示例**：

```css
.container {
  display: flex;
  flex-flow: row wrap; /* 水平排列 + 换行 */
}
```

### 4. `justify-content`：项目在主轴上的对齐方式

定义项目在主轴上的空间分配和对齐。

| 属性值              | 说明                            |
| ---------------- | ----------------------------- |
| `flex-start`（默认） | 项目向主轴起点对齐。                    |
| `flex-end`       | 项目向主轴终点对齐。                    |
| `center`         | 项目在主轴上居中对齐。                   |
| `space-between`  | 项目两端对齐，项目之间的间距相等（两端无间距）。      |
| `space-around`   | 每个项目两侧的间距相等（项目之间的间距是两端的 2 倍）。 |
| `space-evenly`   | 项目之间和两端的间距都相等。                |
**示例**：

```css
.container {
  display: flex;
  justify-content: space-between; /* 两端对齐，项目间距相等 */
}
```

### 5. `align-items`：项目在交叉轴上的对齐方式

定义项目在交叉轴上的对齐（适用于**单行项目**）。

| 属性值           | 说明                              |
| ------------- | ------------------------------- |
| `stretch`（默认） | 若项目未设置高度（或高度为 `auto`），则拉伸至容器高度。 |
| `flex-start`  | 项目向交叉轴起点对齐。                     |
| `flex-end`    | 项目向交叉轴终点对齐。                     |
| `center`      | 项目在交叉轴上居中对齐（常用：垂直居中）。           |
| `baseline`    | 项目的第一行文字基线对齐。                   |
**示例**：

```css
.container {
  display: flex;
  align-items: center; /* 项目垂直居中 */
}
```

### 6. `align-content`：多根轴线的对齐方式

定义**多行项目**（即 `flex-wrap: wrap` 时）在交叉轴上的对齐（单行项目无效）。

| 属性值             | 说明               |
| --------------- | ---------------- |
| `stretch`（默认）   | 轴线拉伸占满交叉轴剩余空间。   |
| `flex-start`    | 轴线向交叉轴起点对齐。      |
| `flex-end`      | 轴线向交叉轴终点对齐。      |
| `center`        | 轴线在交叉轴上居中对齐。     |
| `space-between` | 轴线两端对齐，轴线之间间距相等。 |
| `space-around`  | 每个轴线两侧间距相等。      |
| `space-evenly`  | 轴线之间和两端间距都相等。    |
**示例**：

```css
.container {
  display: flex;
  flex-wrap: wrap;
  align-content: center; /* 多行项目垂直居中 */
}
```

## 三、Flex 项目的属性（6 个核心属性）

### 1. `order`：项目的排列顺序

定义项目的排列顺序，数值越小越靠前（默认值为 `0`），可以是负数。

**示例**：

```css
.item1 { order: 2; } /* 排第 3（假设其他项目 order 为 0、1） */
.item2 { order: -1; } /* 排第 1 */
.item3 { order: 0; } /* 排第 2 */
```

### 2. `flex-grow`：项目的放大比例

定义当主轴有剩余空间时，项目的放大比例（默认值为 `0`，即不放大）。

- 若所有项目的 `flex-grow` 都为 `1`，则它们**平分剩余空间**。
- 若一个项目的 `flex-grow` 为 `2`，其他为 `1`，则它占的空间是其他的 **2 倍**。

**示例**：

```css
.item { flex-grow: 1; } /* 所有项目平分剩余空间 */
```

### 3. `flex-shrink`：项目的缩小比例

定义当主轴空间不足时，项目的缩小比例（默认值为 `1`，即缩小）。

- 若所有项目的 `flex-shrink` 都为 `1`，则空间不足时**等比例缩小**。
- 若一个项目的 `flex-shrink` 为 `0`，其他为 `1`，则空间不足时**它不缩小**。

**示例**：

```css
.item1 { flex-shrink: 0; } /* 空间不足时不缩小 */
.item2 { flex-shrink: 1; } /* 空间不足时缩小 */
```

### 4. `flex-basis`：项目在主轴上的初始大小

定义项目在主轴上的初始大小（默认值为 `auto`，即项目的本来大小），可以设置具体数值（如 `200px`）。

**示例**：

```css
.item { flex-basis: 200px; } /* 项目在主轴上的初始大小为 200px */
```

### 5. `flex`：`flex-grow` + `flex-shrink` + `flex-basis` 的简写

合并设置项目的放大、缩小和初始大小，默认值为 `0 1 auto`。

**常用简写**：

- `flex: 1` → 等价于 `1 1 0%`（项目放大缩小，初始大小为 0，平分空间）。
- `flex: auto` → 等价于 `1 1 auto`（项目放大缩小，初始大小为本身）。
- `flex: none` → 等价于 `0 0 auto`（项目不放大不缩小）。

**示例**：

```css
.item { flex: 1; } /* 项目平分容器空间 */
```

### 6. `align-self`：单个项目在交叉轴上的对齐方式

覆盖容器的 `align-items` 属性，仅对**单个项目**生效（默认值为 `auto`，即继承容器的 `align-items`）。

属性值与 `align-items` 一致：`stretch`、`flex-start`、`flex-end`、`center`、`baseline`。

**示例**：

```css
.container {
  display: flex;
  align-items: flex-start; /* 所有项目默认向交叉轴起点对齐 */
}
.item2 {
  align-self: center; /* 仅 item2 垂直居中 */
}
```

## 四、Flex 布局常见应用场景

### 1. 垂直居中（最常用）

```html
<div class="container">
  <div class="item">垂直居中</div>
</div>
```

```css
.container {
  display: flex;
  justify-content: center; /* 水平居中 */
  align-items: center;     /* 垂直居中 */
  height: 300px;
  border: 1px solid #eee;
}
```

### 2. 两栏布局（左侧固定，右侧自适应）

```html
<div class="container">
  <div class="left">左侧固定（200px）</div>
  <div class="right">右侧自适应</div>
</div>
```

```css
.container { display: flex; }
.left { width: 200px; background: #f0f0f0; }
.right { flex: 1; background: #e0e0e0; } /* flex: 1 占满剩余空间 */
```

### 3. 等分布局（3 列等分）

```html
<div class="container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

```css
.container { display: flex; }
.item { flex: 1; border: 1px solid #eee; text-align: center; }
```

### 4. 导航栏（两端对齐，中间间距相等）

```html
<div class="nav">
  <a href="#">首页</a>
  <a href="#">产品</a>
  <a href="#">关于</a>
  <a href="#">联系</a>
</div>
```

```css
.nav {
  display: flex;
  justify-content: space-around; /* 项目两侧间距相等 */
  background: #333;
}
.nav a { color: #fff; padding: 10px 20px; text-decoration: none; }
```

## 五、注意事项

1. **Flex 容器的子元素（项目）**：`float`、`clear`、`vertical-align` 属性会失效（Flex 布局有自己的对齐方式）。
2. **一维布局**：Flex 是**一维布局**（主要处理一行或一列），若需二维布局（行和列同时控制），推荐使用 **Grid 布局**。
3. **兼容性**：现代浏览器（Chrome、Firefox、Safari、Edge）均完美支持，IE 10 及以下需加前缀（如 `-ms-flex`），但目前 IE 已基本淘汰。