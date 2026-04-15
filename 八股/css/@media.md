`@media` 是 CSS3 引入的**核心响应式布局工具**，它能根据设备的**屏幕尺寸、方向、分辨率**等特性，动态应用不同的 CSS 样式，实现 “一套代码，多端适配”。
## 一、核心概念

媒体查询的本质是 **“条件判断”**：

> 当设备满足指定的条件时，应用 `{}` 内的样式；不满足时，忽略这些样式。

它是响应式网页设计（RWD）的三大核心技术之一（另外两个是弹性网格、弹性媒体）。

## 二、基本语法

```css
@media 媒体类型 and (媒体特性) {
  /* 满足条件时应用的 CSS 样式 */
  selector {
    property: value;
  }
}
```

### 语法拆解

|部分|作用|示例|
|---|---|---|
|`@media`|声明媒体查询的关键字|固定写法|
|媒体类型|指定设备类型（可选，默认 `all`）|`screen`（屏幕）、`print`（打印）|
|`and`|逻辑运算符，连接多个条件|需同时满足媒体类型和媒体特性|
|媒体特性|具体的判断条件（核心）|`(min-width: 768px)`|
|`{}`|满足条件时应用的样式块|普通 CSS 规则|

## 三、常用媒体类型

现在开发中**最常用 `all`**（默认值，适用于所有设备），但仍需了解其他类型：

|媒体类型|适用场景|说明|
|---|---|---|
|`all`|所有设备|开发中默认使用，可省略不写|
|`screen`|电脑、手机、平板屏幕|传统响应式布局常用，现在基本被 `all` 替代|
|`print`|打印机 / 打印预览|用于定制打印样式（隐藏导航、调整字体等）|
|`speech`|屏幕阅读器|为视障用户优化语音播报样式|

## 四、核心媒体特性（最常用）

媒体特性是 `@media` 的灵魂，以下是开发中 90% 场景会用到的特性：

### 1. 尺寸相关（最核心）

|特性|含义|示例|说明|
|---|---|---|---|
|`width`|视口宽度|`(width: 768px)`|精确等于 768px（极少用）|
|`min-width`|视口**最小**宽度（≥）|`(min-width: 768px)`|视口宽度 ≥ 768px 时生效（移动优先核心）|
|`max-width`|视口**最大**宽度（≤）|`(max-width: 1024px)`|视口宽度 ≤ 1024px 时生效（桌面优先核心）|
|`height`|视口高度|同宽度逻辑，极少用||
|`min-height` / `max-height`|视口最小 / 最大高度|同宽度逻辑，极少用||

> **关键记忆**：
> 
> - `min-width`：“大于等于”，从小到大写（移动优先）
> - `max-width`：“小于等于”，从大到小写（桌面优先）

### 2. 方向相关

|特性|含义|可选值|
|---|---|---|
|`orientation`|设备方向|`portrait`（竖屏，高度 > 宽度）、`landscape`（横屏，宽度 > 高度）|

### 3. 分辨率相关

|特性|含义|示例|
|---|---|---|
|`resolution`|设备分辨率|`(min-resolution: 2dppx)`（Retina 屏，2 倍像素密度）|
|`min-resolution` / `max-resolution`|最小 / 最大分辨率|同宽度逻辑|

> 单位说明：`dppx`（每像素点的物理像素数）是现代推荐单位，1dppx = 96dpi。

### 4. 宽高比相关

|特性|含义|示例|
|---|---|---|
|`aspect-ratio`|视口宽高比|`(aspect-ratio: 16/9)`（标准宽屏）|
|`min-aspect-ratio` / `max-aspect-ratio`|最小 / 最大宽高比|`(min-aspect-ratio: 16/9)`（宽屏及以上）|

## 五、逻辑运算符

用于组合多个条件，实现复杂判断：

### 1. `and`：同时满足所有条件

```css
/* 屏幕宽度 ≥768px 且 ≤1024px（平板） */
@media (min-width: 768px) and (max-width: 1024px) {
  body { font-size: 16px; }
}
```

### 2. `,`（逗号）：满足任一条件即可（逻辑 “或”）

```css
/* 宽度 ≥768px 或 横屏模式 */
@media (min-width: 768px), (orientation: landscape) {
  .container { padding: 20px; }
}
```

### 3. `not`：否定整个查询

```css
/* 除了打印设备外的所有设备 */
@media not print {
  .no-print { display: block; }
}
```

### 4. `only`：兼容旧浏览器（现代开发可省略）

旧浏览器不支持媒体查询时，会忽略 `only` 开头的查询，避免应用错误样式：

```css
@media only screen and (min-width: 768px) {
  /* 现代浏览器正常应用，旧浏览器忽略 */
}
```

## 六、移动优先 vs 桌面优先

这是使用 `@media` 的两种核心策略，**推荐优先使用移动优先**。

### 1. 移动优先（Mobile First）

- **思路**：先写小屏幕（手机）的默认样式，再用 `min-width` 逐步适配大屏幕（平板、桌面）
- **优点**：优先保证移动设备体验（符合移动互联网趋势），代码更简洁，性能更优
- **示例**：
    
```css
    /* 默认样式：手机（<768px） */
    body { font-size: 14px; }
    .grid { display: grid; grid-template-columns: 1fr; } /* 1列 */
    
    /* 平板（≥768px） */
    @media (min-width: 768px) {
      body { font-size: 16px; }
      .grid { grid-template-columns: 1fr 1fr; } /* 2列 */
    }
    
    /* 桌面（≥1024px） */
    @media (min-width: 1024px) {
      .grid { grid-template-columns: 1fr 1fr 1fr; } /* 3列 */
    }
```

### 2. 桌面优先（Desktop First）

- **思路**：先写大屏幕（桌面）的默认样式，再用 `max-width` 逐步适配小屏幕
- **适用场景**：传统网站改造，或核心用户以桌面端为主
- **示例**：
    
```css
    /* 默认样式：桌面（>1024px） */
    .grid { display: grid; grid-template-columns: 1fr 1fr 1fr; } /* 3列 */
    
    /* 平板（≤1024px） */
    @media (max-width: 1024px) {
      .grid { grid-template-columns: 1fr 1fr; } /* 2列 */
    }
    
    /* 手机（≤768px） */
    @media (max-width: 768px) {
      .grid { grid-template-columns: 1fr; } /* 1列 */
    }
```


## 七、常见断点（Breakpoint）

断点是指触发样式变化的视口宽度，**建议根据内容调整，而非固定设备尺寸**。以下是工业界常用的参考断点：

|断点|适用设备|说明|
|---|---|---|
|`320px`|小屏手机（iPhone SE）|极少单独写，默认样式通常覆盖|
|`480px`|大屏手机（iPhone 15 Pro）|调整字体、间距|
|`768px`|平板（iPad）|从 1 列变 2 列，显示更多内容|
|`1024px`|小桌面（笔记本）|从 2 列变 3 列，恢复完整导航|
|`1200px` / `1440px`|大桌面（显示器）|限制最大宽度，避免内容过宽|

> **最佳实践**：当内容 “看起来不舒服” 时（比如文本行宽超过 80 字符、图片过大），再添加断点，不要为了设备而设备。

## 八、实际应用场景

### 1. 响应式网格布局

```css
/* 移动优先：1列 */
.product-grid {
  display: grid;
  gap: 20px;
  grid-template-columns: 1fr;
}

/* 平板：2列 */
@media (min-width: 768px) {
  .product-grid { grid-template-columns: 1fr 1fr; }
}

/* 桌面：3列 */
@media (min-width: 1024px) {
  .product-grid { grid-template-columns: 1fr 1fr 1fr; }
}
```

### 2. 打印样式优化

```css
/* 打印时隐藏导航、侧边栏，调整字体 */
@media print {
  .nav, .sidebar, .footer { display: none; }
  body { font-size: 12pt; color: black; }
  .content { width: 100%; margin: 0; }
}
```

### 3. 横屏 / 竖屏适配

```css
/* 竖屏：图片占满宽度 */
@media (orientation: portrait) {
  .hero-image { width: 100%; height: auto; }
}

/* 横屏：图片占满高度 */
@media (orientation: landscape) {
  .hero-image { height: 100vh; width: auto; }
}
```
### 4. Retina 屏高清图片

```css
/* 普通屏：1倍图 */
.logo { background-image: url('logo.png'); }

/* Retina 屏：2倍图 */
@media (min-resolution: 2dppx) {
  .logo { background-image: url('logo@2x.png'); background-size: contain; }
}
```

## 九、最佳实践

1. **必须配合 Viewport Meta 标签**
    
    移动端浏览器默认会缩放页面，需强制视口与设备宽度一致：
    
```html
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
```
    
2. **优先使用移动优先**
    
    从简单到复杂，代码更易维护，性能更优。
    
3. **使用相对单位**
    
    配合 `em`、`rem`、`%`、`fr` 等相对单位，实现更灵活的适配。
    
4. **避免过度使用媒体查询**
    
    能用 Flexbox/Grid 自适应解决的问题，就不要用媒体查询。
    
5. **测试真实设备**
    
    浏览器开发者工具的模拟功能不能完全替代真实设备，需在真机上测试。
    
6. **保持代码组织**
    
    将媒体查询放在对应默认样式的附近，或统一放在文件末尾，方便维护。

## 十、总结

`@media` 是响应式布局的核心，通过 “条件判断” 让网页适应不同设备。掌握以下几点即可应对 99% 的场景：

- 核心特性：`min-width`、`max-width`、`orientation`
- 核心策略：移动优先（`min-width` 从小到大）
- 关键配合：Viewport Meta 标签
- 最佳实践：断点基于内容，而非设备