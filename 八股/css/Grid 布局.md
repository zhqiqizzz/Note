Grid 布局（CSS Grid Layout）是 CSS3 引入的**二维布局模型**，可以同时控制 “行” 和 “列” 两个维度，是目前解决复杂页面布局（如仪表盘、多列网格、响应式卡片）的最强大工具，与 Flex 布局（一维）配合使用可覆盖 99% 的布局需求。

## 一、核心概念

Grid 布局由 **Grid 容器（Grid Container）**、**Grid 项目（Grid Items）** 和**网格线 / 轨道 / 单元格 / 区域**组成：

|概念|说明|
|---|---|
|**Grid 容器**|设置了 `display: grid` 或 `display: inline-grid` 的元素，是 Grid 项目的父元素。|
|**Grid 项目**|Grid 容器的**直接子元素**（孙子元素不受影响）。|
|**网格线**|划分网格的线，分为 “行网格线” 和 “列网格线”，从 1 开始编号（也可用负数从后往前数）。|
|**网格轨道**|两条相邻网格线之间的空间，即 “行” 或 “列”。|
|**网格单元格**|两条相邻行网格线和两条相邻列网格线之间的空间，是 Grid 布局的**最小单位**。|
|**网格区域**|由多个网格单元格组成的矩形区域（可以是单个单元格）。|

## 二、Grid 容器的属性（10 个核心属性）

### 1. `display`：定义 Grid 容器

|属性值|说明|
|---|---|
|`grid`|块级 Grid 容器（独占一行）。|
|`inline-grid`|行内级 Grid 容器（宽度由内容决定，可与其他元素在同一行）。|
**示例**：

```css
.container {
  display: grid; /* 定义为 Grid 容器 */
}
```

### 2. `grid-template-columns` / `grid-template-rows`：定义显式网格的列和行

用空格分隔的值定义每一列 / 行的大小，值的数量就是列 / 行的数量。

#### 常用单位：

- **固定单位**：`px`、`em`、`rem`（如 `200px`）。
- **百分比**：`%`（相对于容器宽度 / 高度）。
- **`fr` 单位**：Grid 专用单位，表示 “剩余空间的比例”（类似 Flex 的 `flex-grow`）。
- **`auto`**：由内容或剩余空间决定大小。
- **`minmax(min, max)`**：定义大小范围（如 `minmax(100px, 1fr)` 表示最小 100px，最大占 1fr）。
- **`repeat(count, size)`**：重复定义（如 `repeat(3, 1fr)` 等价于 `1fr 1fr 1fr`）。
- **`auto-fill` / `auto-fit`**：配合 `repeat` 使用，自动填充尽可能多的列 / 行（`auto-fill` 会保留空轨道，`auto-fit` 会合并空轨道，响应式常用）。

**示例**：

```css
.container {
  display: grid;
  /* 3 列：第 1 列 200px，第 2 列 1fr，第 3 列 1fr */
  grid-template-columns: 200px 1fr 1fr;
  /* 2 行：第 1 行 100px，第 2 行最小 50px、最大占剩余空间 */
  grid-template-rows: 100px minmax(50px, auto);
  /* 等价写法：repeat(3, 1fr) → 3 列，每列 1fr */
  grid-template-columns: repeat(3, 1fr);
  /* 响应式：自动填充，每列最小 200px，最大 1fr */
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
}
```

### 3. `grid-template-areas`：定义网格区域

用 “字符串” 定义网格区域，配合项目的 `grid-area` 使用，直观易读。

- 每一行用一个字符串表示，字符串内的空格分隔列。
- 相同的名称表示合并为一个区域。
- `.` 表示空单元格。

**示例**：

```css
.container {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-rows: 60px 1fr 60px;
  /* 定义 3 行 2 列的区域 */
  grid-template-areas:
    "header header"  /* 第 1 行：header 占 2 列 */
    "sidebar main"   /* 第 2 行：sidebar 占第 1 列，main 占第 2 列 */
    "footer footer"; /* 第 3 行：footer 占 2 列 */
}
```

### 4. `gap` / `row-gap` / `column-gap`：定义网格间距

- `row-gap`：行间距。
- `column-gap`：列间距。
- `gap`：`row-gap` + `column-gap` 的简写（若只写一个值，则行 / 列间距相同）。

**示例**：

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px; /* 行间距和列间距都是 20px */
  /* 等价写法：
  row-gap: 20px;
  column-gap: 20px;
  */
}
```

### 5. `justify-items` / `align-items`：项目在单元格内的对齐

- `justify-items`：项目在单元格内的**水平**对齐。
- `align-items`：项目在单元格内的**垂直**对齐。
- `place-items`：`align-items` + `justify-items` 的简写（若只写一个值，则水平 / 垂直对齐相同）。

**属性值**：

|属性值|说明|
|---|---|
|`stretch`（默认）|项目拉伸占满单元格（若未设置宽 / 高）。|
|`start`|项目向单元格起点对齐（左 / 上）。|
|`end`|项目向单元格终点对齐（右 / 下）。|
|`center`|项目在单元格内居中对齐。|
**示例**：

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  justify-items: center; /* 项目水平居中 */
  align-items: center;   /* 项目垂直居中 */
  /* 等价写法：place-items: center; */
}
```

### 6. `justify-content` / `align-content`：整个网格在容器内的对齐

当网格总大小小于容器大小时，控制整个网格在容器内的对齐。

- `justify-content`：整个网格的**水平**对齐。
- `align-content`：整个网格的**垂直**对齐。
- `place-content`：`align-content` + `justify-content` 的简写。

**属性值**（与 Flex 的 `justify-content` 类似）：

| 属性值             | 说明               |
| --------------- | ---------------- |
| `start`         | 网格向容器起点对齐。       |
| `end`           | 网格向容器终点对齐。       |
| `center`        | 网格在容器内居中对齐。      |
| `space-between` | 网格两端对齐，轨道之间间距相等。 |
| `space-around`  | 每个轨道两侧间距相等。      |
| `space-evenly`  | 轨道之间和两端间距都相等。    |
**示例**：

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 200px); /* 网格总宽度 600px，小于容器宽度 */
  justify-content: center; /* 整个网格水平居中 */
}
```

### 7. `grid-auto-columns` / `grid-auto-rows`：定义隐式网格的大小

当项目数量超过显式网格的单元格数量，或项目位置超出显式网格时，会自动生成**隐式网格**，这两个属性控制隐式网格的列 / 行大小。

**示例**：

```css
.container {
  display: grid;
  grid-template-columns: repeat(2, 1fr); /* 显式 2 列 */
  grid-template-rows: 100px; /* 显式 1 行 */
  grid-auto-rows: 80px; /* 隐式行的高度为 80px */
}
```

### 8. `grid-auto-flow`：控制项目的自动放置方向

定义项目在没有指定位置时的自动放置顺序。

|属性值|说明|
|---|---|
|`row`（默认）|先填满一行，再换行（“先行后列”）。|
|`column`|先填满一列，再换列（“先列后行”）。|
|`row dense`|先行后列，且尽可能填充空白（“密集排列”）。|
|`column dense`|先列后行，且尽可能填充空白。|
**示例**：

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-auto-flow: column; /* 先列后行放置项目 */
}
```

### 9. `grid`：容器属性的简写

合并 `grid-template-rows`、`grid-template-columns`、`grid-template-areas`、`grid-auto-rows`、`grid-auto-columns`、`grid-auto-flow`，但语法较复杂，不推荐新手使用。

## 三、Grid 项目的属性（6 个核心属性）

### 1. `grid-column` / `grid-row`：用网格线定位项目

- `grid-column`：`grid-column-start` + `grid-column-end` 的简写，定义项目的**水平**位置。
- `grid-row`：`grid-row-start` + `grid-row-end` 的简写，定义项目的**垂直**位置。

**写法**：

- `grid-column: 1 / 3` → 从第 1 条列网格线开始，到第 3 条列网格线结束（占 2 列）。
- `grid-column: 1 / span 2` → 从第 1 条列网格线开始，跨 2 列（等价于 `1 / 3`）。
- `grid-column: -1 / -3` → 从倒数第 1 条列网格线开始，到倒数第 3 条列网格线结束。

**示例**：

```css
.item1 {
  grid-column: 1 / 3; /* 占第 1-2 列（跨 2 列） */
  grid-row: 1 / 2;    /* 占第 1 行 */
}
.item2 {
  grid-column: 2 / span 2; /* 从第 2 列开始，跨 2 列 */
  grid-row: 2 / 4;         /* 占第 2-3 行（跨 2 行） */
}
```

### 2. `grid-area`：用网格区域或网格线定位项目

- 配合容器的 `grid-template-areas` 使用：直接写区域名称。
- 也可作为 `grid-row-start` / `grid-column-start` / `grid-row-end` / `grid-column-end` 的简写。

**示例**（配合 `grid-template-areas`）：

```css
/* 容器 */
.container {
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
}

/* 项目 */
.header { grid-area: header; } /* 放在 header 区域 */
.sidebar { grid-area: sidebar; } /* 放在 sidebar 区域 */
.main { grid-area: main; } /* 放在 main 区域 */
.footer { grid-area: footer; } /* 放在 footer 区域 */
```

### 3. `justify-self` / `align-self`：单个项目在单元格内的对齐

覆盖容器的 `justify-items` / `align-items`，仅对**单个项目**生效。

- `justify-self`：单个项目的**水平**对齐。
- `align-self`：单个项目的**垂直**对齐。
- `place-self`：`align-self` + `justify-self` 的简写。

**属性值**与 `justify-items` / `align-items` 一致：`stretch`、`start`、`end`、`center`。

**示例**：

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  justify-items: start; /* 所有项目默认水平左对齐 */
}
.item2 {
  justify-self: center; /* 仅 item2 水平居中 */
  align-self: center;   /* 仅 item2 垂直居中 */
}
```

### 4. `order`：项目的排列顺序

定义项目的排列顺序，数值越小越靠前（默认值为 `0`），可以是负数（与 Flex 的 `order` 用法一致）。

**示例**：

```css
.item1 { order: 2; }
.item2 { order: -1; } /* 排最前 */
.item3 { order: 0; }
```

## 四、Grid 布局常见应用场景

### 1. 圣杯布局（Header + Sidebar + Main + Footer）

```html
<div class="container">
  <div class="header">Header</div>
  <div class="sidebar">Sidebar</div>
  <div class="main">Main</div>
  <div class="footer">Footer</div>
</div>
```

```css
.container {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-rows: 60px 1fr 60px;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  min-height: 100vh;
  gap: 10px;
}
.header { grid-area: header; background: #333; color: #fff; }
.sidebar { grid-area: sidebar; background: #f0f0f0; }
.main { grid-area: main; background: #e0e0e0; }
.footer { grid-area: footer; background: #333; color: #fff; }
```

### 2. 响应式卡片布局（自动填充）

```html
<div class="container">
  <div class="card">卡片 1</div>
  <div class="card">卡片 2</div>
  <div class="card">卡片 3</div>
  <div class="card">卡片 4</div>
  <div class="card">卡片 5</div>
</div>
```

```css
.container {
  display: grid;
  /* 响应式：自动填充，每列最小 250px，最大 1fr */
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 20px;
  padding: 20px;
}
.card {
  background: #f0f0f0;
  padding: 20px;
  border-radius: 8px;
  text-align: center;
}
```

### 3. 九宫格布局

```html
<div class="container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
  <div class="item">4</div>
  <div class="item">5</div>
  <div class="item">6</div>
  <div class="item">7</div>
  <div class="item">8</div>
  <div class="item">9</div>
</div>
```

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 100px);
  grid-template-rows: repeat(3, 100px);
  gap: 10px;
  justify-content: center;
}
.item {
  background: #333;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 24px;
}
```

## 五、注意事项

1. **Grid vs Flex**：
    
    - **Grid**：二维布局（同时控制行和列），适合复杂的页面结构、多列网格。
    - **Flex**：一维布局（控制一行或一列），适合简单的对齐、空间分配、一维排列。
    - 两者可配合使用（如 Grid 控制整体页面结构，Flex 控制局部组件内的排列）。
    
2. **Grid 项目的失效属性**：
    
    - `float`、`clear`、`vertical-align` 对 Grid 项目失效（Grid 有自己的对齐方式）。
    
3. **兼容性**：
    
    - 现代浏览器（Chrome、Firefox、Safari、Edge）均完美支持。
    - IE 11 仅支持部分旧语法（需加 `-ms-` 前缀），但目前 IE 已基本淘汰，无需考虑。