CSS **定位（Position）** 是控制元素在页面中位置的核心属性，通过 `position` 配合 `top`/`right`/`bottom`/`left` 实现灵活的布局，是前端开发的必备技能。

以下是 5 种常见定位类型的详细介绍，附核心特点、用法示例和注意事项。
## 一、定位的基础：5 种类型

|定位类型|属性值|是否脱离文档流|定位基准|典型场景|
|---|---|---|---|---|
|**静态定位**|`static`（默认）|否|正常文档流（无特殊定位）|默认布局，无需额外设置|
|**相对定位**|`relative`|否|元素**自身原来的位置**|微调位置、作为绝对定位的祖先|
|**绝对定位**|`absolute`|是|最近的**非 static 祖先元素**|弹窗、悬浮元素、容器内定位|
|**固定定位**|`fixed`|是|**浏览器视口（Viewport）**|固定导航、回到顶部按钮|
|**粘性定位**|`sticky`|否（滚动前）|结合 `relative` + `fixed`|吸顶导航、吸底按钮|

## 二、各定位类型详解

### 1. 静态定位（`static`，默认）

- **特点**：元素按正常文档流排列，`top`/`right`/`bottom`/`left`/`z-index` 均无效。
- **用法**：无需额外设置，所有元素默认都是 `static`。
    
    
    
```css
    .box {
      position: static; /* 默认值，可省略 */
    }
```
### 2. 相对定位（`relative`）

- **核心特点**：
    
    - 相对于**自身原来的位置**移动（“相对自己”）。
    - **不脱离文档流**：原来的位置仍保留，其他元素不会填补。
    
- **用法示例**：
	
```
    .box {
      position: relative;
      top: 20px;    /* 相对于原位置向下移动 20px */
      left: 30px;   /* 相对于原位置向右移动 30px */
    }
```

- **典型场景**：
    
    - 微调元素位置（如让图标向下对齐文字）。
    - 作为**绝对定位元素的 “定位祖先”**（见下）。
### 3. 绝对定位（`absolute`）

- **核心特点**：
    
    - 相对于**最近的非 static 祖先元素**定位（若没有，则相对于 `<body>`）。
    - **脱离文档流**：原来的位置不保留，其他元素会填补。
    - 宽度 / 高度默认由内容决定（可手动设置）。
    
- **关键规则**：“父相子绝”—— 父元素设 `relative`，子元素设 `absolute`，是最常用的组合。
- **用法示例**：
    
```html
    <div class="parent">
      <div class="child">绝对定位元素</div>
    </div>
```
    
```css
    .parent {
      position: relative; /* 父元素设为相对定位，作为定位基准 */
      width: 300px;
      height: 200px;
      border: 1px solid #000;
    }
    .child {
      position: absolute;
      top: 10px;    /* 距离父元素顶部 10px */
      right: 10px;  /* 距离父元素右侧 10px */
      width: 80px;
      height: 40px;
      background: red;
    }
```
    
- **典型场景**：
    
    - 弹窗的关闭按钮（定位在弹窗右上角）。
    - 图片上的标签（如 “新品”“热卖”）。
    - 悬浮菜单、下拉框。
### 4. 固定定位（`fixed`）

- **核心特点**：
    
    - 相对于**浏览器视口**定位（无论页面如何滚动，元素位置不变）。
    - **脱离文档流**：原来的位置不保留。
    
- **用法示例**：
	
```css
    .header {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      background: #333;
      color: #fff;
      padding: 10px;
    }
```
    
- **典型场景**：
    
    - 固定顶部导航栏。
    - 回到顶部按钮（固定在右下角）。
    - 悬浮广告、客服弹窗。
### 5. 粘性定位（`sticky`，CSS3 新增）

- **核心特点**：
    
    - 结合 `relative` + `fixed`：滚动前是 `relative`（正常文档流），滚动到**指定阈值**（如 `top: 0`）时变成 `fixed`（固定）。
    - **不脱离文档流**：原来的位置始终保留。
    
- **触发条件**（缺一不可）：
    
    1. 必须设置 `top`/`right`/`bottom`/`left` 中的一个（通常是 `top`）。
    2. 父元素不能有 `overflow: hidden`/`auto`/`scroll`（否则会失效）。
    3. 元素必须在父元素的可视区域内（父元素高度要大于元素高度）。
    
- **用法示例**：
    
```css
    .nav {
      position: sticky;
      top: 0; /* 滚动到距离视口顶部 0px 时固定 */
      background: #fff;
      padding: 10px;
      border-bottom: 1px solid #eee;
    }
```
    
- **典型场景**：
    
    - 吸顶导航（滚动时固定在顶部）。
    - 列表的分组标题（如通讯录的字母索引）。
    - 吸底提交按钮。

## 三、层叠顺序：`z-index`

当多个定位元素重叠时，用 `z-index` 控制堆叠顺序：

- **规则**：`z-index` 值越大，元素越靠上（越先被看到）。
- **前提**：仅对**非 static 定位元素**有效（`relative`/`absolute`/`fixed`/`sticky`）。
- **层叠上下文**：父元素的 `z-index` 会影响子元素 —— 若父元素 `z-index` 低，子元素 `z-index` 再高也不会超过父元素的兄弟元素。
- **用法示例**：
    
```css
    .box1 {
      position: relative;
      z-index: 1;
      background: red;
      width: 100px;
      height: 100px;
    }
    .box2 {
      position: relative;
      z-index: 2; /* 值更大，靠上 */
      background: blue;
      width: 100px;
      height: 100px;
      margin-top: -50px; /* 与 box1 重叠 */
    }
```

## 四、常见误区与注意事项

1. **绝对定位找不到祖先**：若绝对定位元素的所有祖先都是 `static`，则会相对于 `<body>` 定位。
2. **粘性定位失效**：检查父元素是否有 `overflow`，是否设置了 `top` 等阈值，父元素高度是否足够。
3. **`z-index` 无效**：确保元素是定位元素（非 `static`），且没有被父元素的层叠上下文限制。
4. **固定定位被遮挡**：检查是否有其他元素 `z-index` 更高，或父元素有 `transform`（会改变固定定位的基准）。

## 五、总结：如何选择定位？

|需求|推荐定位类型|
|---|---|
|正常布局，无需特殊定位|`static`|
|微调元素位置|`relative`|
|定位在某个容器内|`absolute`（父元素 `relative`）|
|固定在视口，滚动不动|`fixed`|
|滚动到指定位置后固定|`sticky`|
