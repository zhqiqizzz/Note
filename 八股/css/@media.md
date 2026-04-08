`@media` 是 CSS 中的**媒体查询规则**，是实现响应式设计的核心工具。它允许根据设备的特性（如屏幕宽度、高度、方向、分辨率等）来条件性地应用 CSS 样式，使网页在不同设备（手机、平板、桌面等）上都能自适应显示。
### 基本语法

```css
@media 媒体类型 and (媒体特性) {
  /* 满足条件时应用的 CSS 样式 */
}
```

### 核心组成部分

#### 1. 媒体类型

指定目标设备类型，常用值：

- `all`：所有设备（默认值，可省略）
- `screen`：屏幕设备（手机、平板、电脑等）
- `print`：打印设备（打印预览或打印时）

#### 2. 媒体特性

描述设备的具体特征，常用特性：

|特性|说明|示例|
|---|---|---|
|`width`|视口宽度|`width: 768px`|
|`min-width`|最小视口宽度（≥ 该值）|`min-width: 768px`|
|`max-width`|最大视口宽度（≤ 该值）|`max-width: 768px`|
|`height`|视口高度|`height: 1024px`|
|`orientation`|屏幕方向|`orientation: portrait`（纵向）/ `landscape`（横向）|
|`resolution`|设备分辨率|`resolution: 2dppx`（Retina 屏）|

### 常用逻辑运算符

- `and`：同时满足多个条件
- `,`（逗号）：满足任意一个条件（逻辑 “或”）
- `not`：否定某个条件（需配合媒体类型使用）
- `only`：仅在完全匹配时应用（用于兼容旧浏览器）

### 实际应用示例

#### 1. 移动端优先的响应式布局

```css
/* 基础样式：默认适配移动端 */
.container {
  width: 100%;
  padding: 10px;
}

/* 平板及以上（≥ 768px） */
@media (min-width: 768px) {
  .container {
    width: 750px;
    margin: 0 auto;
  }
}

/* 桌面端（≥ 1200px） */
@media (min-width: 1200px) {
  .container {
    width: 1170px;
  }
}
```

#### 2. 打印样式优化

```css
@media print {
  /* 隐藏导航、广告等不必要元素 */
  .nav, .ad {
    display: none;
  }
  /* 调整内容宽度适配纸张 */
  .content {
    width: 100%;
  }
}
```

#### 3. 屏幕方向适配

```css
/* 纵向屏幕 */
@media (orientation: portrait) {
  .box {
    flex-direction: column;
  }
}

/* 横向屏幕 */
@media (orientation: landscape) {
  .box {
    flex-direction: row;
  }
}
```

### 关键注意事项

1. **移动端优先**：建议先写移动端基础样式，再用 `min-width` 逐步适配更大屏幕，代码更简洁。
2. **样式覆盖**：媒体查询中的样式会覆盖基础样式（需保证选择器优先级相同或更高）。
3. **单位选择**：优先使用 `em`、`rem` 或百分比，避免固定像素值，增强适配性。