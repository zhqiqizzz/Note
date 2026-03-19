要彻底理解 **模板 → AST → 静态标记 → render 函数 → 虚拟 DOM** 这一整套流程，最经典的参考是 **Vue 框架的编译与渲染原理**。我们可以把这套流程拆分为 **「编译阶段」**（把用户写的模板变成可执行的代码）和 **「运行时阶段」**（执行代码生成虚拟 DOM 并更新页面），一步步拆解讲解。

## 一、整体流程概览

先看一张完整的流程图（以 Vue 3 为例），建立全局认知：

```plaintext
用户编写的 Template（模板）
    ↓
【编译阶段】
    ↓
1. 解析（Parse）：模板 → AST（抽象语法树）
    ↓
2. 优化（Transform）：AST → 带静态标记的 AST
    ↓
3. 代码生成（Codegen）：带标记的 AST → render 函数
    ↓
【运行时阶段】
    ↓
4. 执行 render 函数：返回 虚拟 DOM（VNode 树）
    ↓
5. 虚拟 DOM Diff：对比新旧 VNode 树，找出最小变化
    ↓
6. 真实 DOM 更新：把变化应用到页面
```

> 补充： 首次执行 `renderEffect` 时，会调用 `component.render()`，构建出 [虚拟 DOM](虚拟%20DOM.md) 
## 二、详细拆解每个环节

### 1. 模板（Template）：用户友好的声明式语法

**定义**：Vue 模板是一种 **HTML 扩展语法**，允许你用类似写 HTML 的方式声明式地描述页面结构，同时嵌入插值表达式（`{{ }}`）、指令（`v-if`/`v-for`）等动态逻辑。

**例子**：

```html
<!-- 这是一个典型的 Vue 模板 -->
<div id="app">
  <h1>静态标题</h1>
  <p>{{ message }}</p>
  <button @click="count++">点击次数: {{ count }}</button>
</div>
```

**特点**：

- 对开发者友好，直观易读；
- 浏览器**不认识**这种语法，必须经过编译转换成 JS 代码才能执行。

### 2. AST（抽象语法树）：模板的结构化表示

**定义**：AST（Abstract Syntax Tree，抽象语法树）是用 **JS 对象树形结构** 来描述模板语法的一种数据结构。它把模板的标签、属性、文本、插值表达式等都拆解成树的节点，方便后续处理。

#### 生成过程：解析（Parse）

编译的第一步是「词法分析 + 语法分析」，把模板字符串解析成 AST：

- **词法分析**：把模板字符串拆成一个个「Token」（比如 `<div`、`id="app"`、`</div>` 等）；
- **语法分析**：根据 Token 之间的嵌套关系，组装成树形结构的 AST。

#### 例子：模板 → AST

还是上面的模板，简化后的 AST 大概长这样：

```javascript
{
  type: 1, // 1 表示元素节点，2 表示文本节点，3 表示插值表达式等
  tag: "div",
  props: [{ name: "id", value: "app" }],
  children: [
    // 子节点 1：静态 h1
    {
      type: 1,
      tag: "h1",
      children: [{ type: 2, content: "静态标题" }]
    },
    // 子节点 2：带插值的 p
    {
      type: 1,
      tag: "p",
      children: [{ type: 3, content: "message" }] // 插值表达式
    },
    // 子节点 3：带事件和插值的 button
    {
      type: 1,
      tag: "button",
      props: [{ name: "onClick", value: "count++" }],
      children: [
        { type: 2, content: "点击次数: " },
        { type: 3, content: "count" }
      ]
    }
  ]
}
```

### 3. 静态标记（Static Marking）：Vue 3 的核心优化

**定义**：静态标记是 Vue 3 编译阶段的**关键优化步骤**。它会遍历 AST，识别出哪些是「静态节点」（永远不会变化的内容），哪些是「动态节点」（会随数据变化的内容），并给动态节点打上「标记（PatchFlags）」，告诉运行时：“只需要关注这些标记的地方，其他的不用管”。
#### 为什么需要静态标记？

在 Vue 2 中，每次数据变化，虚拟 DOM 都会**全量对比**整个树，哪怕大部分节点根本没变化。Vue 3 通过静态标记，实现了 **「靶向更新」**—— 只对比有动态标记的节点，大幅减少了 Diff 的开销。
#### 核心概念：PatchFlags（动态标记）

Vue 3 用枚举值定义了不同类型的动态性，比如：

```javascript
const PatchFlags = {
  TEXT: 1,       // 动态文本
  CLASS: 2,      // 动态 class
  STYLE: 4,      // 动态 style
  PROPS: 8,      // 动态属性
  EVENT: 16,     // 动态事件
  // ... 更多类型
};
```

#### 例子：带静态标记的 AST

还是刚才的模板，经过静态标记后，AST 会变成这样（关键变化）：

```javascript
{
  type: 1,
  tag: "div",
  props: [{ name: "id", value: "app" }],
  children: [
    // 子节点 1：静态 h1（没有任何标记，运行时直接跳过）
    { type: 1, tag: "h1", children: [{ type: 2, content: "静态标题" }] },
    
    // 子节点 2：动态 p（标记 TEXT，因为文本是动态的）
    {
      type: 1,
      tag: "p",
      children: [{ type: 3, content: "message" }],
      patchFlag: 1 // PatchFlags.TEXT，标记“动态文本”
    },
    
    // 子节点 3：动态 button（标记 TEXT + EVENT）
    {
      type: 1,
      tag: "button",
      props: [{ name: "onClick", value: "count++" }],
      children: [
        { type: 2, content: "点击次数: " },
        { type: 3, content: "count" }
      ],
      patchFlag: 1 | 16 // TEXT (1) + EVENT (16)，按位或合并
    }
  ]
}
```

#### 额外优化：静态提升（Static Hoisting）

除了标记，Vue 3 还会把**静态节点直接提升到 render 函数外面**，避免每次执行 render 都重新创建静态节点：

```javascript
// 静态提升前（每次 render 都重新创建 h1）
function render() {
  return createVNode("div", null, [
    createVNode("h1", null, "静态标题"), // 每次都重新创建
    // ... 动态节点
  ]);
}

// 静态提升后（h1 只创建一次，复用）
const _hoisted_1 = createVNode("h1", null, "静态标题"); // 提升到外面
function render() {
  return createVNode("div", null, [
    _hoisted_1, // 直接复用
    // ... 动态节点
  ]);
}
```

### 4. render 函数：把 AST 变成可执行的 JS 代码

**定义**：render 函数是一个**纯 JS 函数**，它的作用是「描述页面应该长什么样」。编译的最后一步，会把带静态标记的 AST 转换成 render 函数代码。

#### 生成过程：代码生成（Codegen）

遍历带标记的 AST，用 `createVNode`（Vue 3 的虚拟节点创建函数，也叫 `h` 函数）把每个节点转换成 JS 调用，最终拼出完整的 render 函数。

#### 例子：带标记的 AST → render 函数

还是上面的模板，最终生成的 render 函数大概长这样（结合了静态提升和 PatchFlags）：

```javascript
import { createVNode as _createVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createBlock as _createBlock } from "vue";

// 静态提升：静态节点只创建一次
const _hoisted_1 = /*#__PURE__*/_createVNode("h1", null, "静态标题", -1 /* HOISTED */);

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", { id: "app" }, [
    _hoisted_1, // 复用静态节点
    _createVNode("p", null, _toDisplayString(_ctx.message), 1 /* TEXT */), // 标记 TEXT
    _createVNode("button", {
      onClick: $event => (_ctx.count++)
    }, "点击次数: " + _toDisplayString(_ctx.count), 1 /* TEXT */) // 标记 TEXT
  ]));
}
```

**关键说明**：

- `_createVNode`：创建虚拟节点的函数，参数是「标签名、属性、子节点、PatchFlags」；
- `1 /* TEXT */`：就是静态标记的 PatchFlags，告诉运行时这个节点只有动态文本；
- `-1 /* HOISTED */`：标记这是静态提升的节点，运行时完全跳过对比。

### 5. 虚拟 DOM（Virtual DOM）：用 JS 对象描述真实 DOM

**定义**：虚拟 DOM（简称 VNode）是一个**用 JS 对象模拟真实 DOM 结构**的轻量级表示。执行 render 函数，就会返回一棵虚拟 DOM 树（VNode 树）。

#### 为什么需要虚拟 DOM？

- **跨平台**：虚拟 DOM 是 JS 对象，不依赖真实 DOM，可以轻松渲染到浏览器、Node.js（SSR）、小程序等不同平台；
- **性能优化**：通过 Diff 算法对比新旧 VNode 树，找出最小变化，再批量更新真实 DOM（减少真实 DOM 操作次数，因为真实 DOM 操作很慢）。

#### 例子：执行 render 函数 → 虚拟 DOM 树

执行上面的 render 函数，返回的虚拟 DOM 树（简化版）大概长这样：

```javascript
// 这就是虚拟 DOM 树（VNode 树）
{
  type: "div",
  props: { id: "app" },
  children: [
    // 静态 h1（复用的）
    { type: "h1", children: "静态标题", patchFlag: -1 },
    // 动态 p
    { 
      type: "p", 
      children: "Hello Vue", // 假设 _ctx.message 是 "Hello Vue"
      patchFlag: 1 // TEXT 标记
    },
    // 动态 button
    { 
      type: "button", 
      props: { onClick: fn },
      children: "点击次数: 0", // 假设 _ctx.count 是 0
      patchFlag: 1 // TEXT 标记
    }
  ]
}
```

### 6. 虚拟 DOM Diff + 真实 DOM 更新：把变化应用到页面

最后一步是**运行时的核心**：当数据变化时（比如 `count` 从 0 变成 1），会触发响应式系统的副作用函数，重新执行 render 函数，生成一棵**新的虚拟 DOM 树**。然后：

#### 步骤 1：Diff 算法对比新旧 VNode 树

Vue 3 的 Diff 算法会利用**静态标记**做「靶向对比」：

- 跳过 `patchFlag: -1` 的静态节点（完全不对比）；
- 只对比有 `patchFlag` 的动态节点，且只对比标记对应的部分（比如 `TEXT` 标记只对比文本内容，不管属性）。

#### 步骤 2：生成最小更新补丁（Patch）

对比后，找出需要更新的地方，生成「补丁」：

```javascript
// 比如 count 从 0 变成 1，补丁大概是：
{
  type: "TEXT",
  node: buttonVNode, // 要更新的 button 节点
  newText: "点击次数: 1" // 新的文本内容
}
```

#### 步骤 3：把补丁应用到真实 DOM

最后，根据补丁，**最小化地操作真实 DOM**，更新页面。

## 三、完整闭环：结合 Proxy 响应式

把之前讲的 Proxy 响应式和今天的内容结合起来，就是 Vue 3 的完整流程：

1. 用户修改数据（比如 `count++`）；
2. Proxy 的 `set` 拦截器触发，通知响应式系统；
3. 响应式系统触发副作用函数（重新执行 render 函数）；
4. 执行 render 函数，生成**新的虚拟 DOM 树**；
5. Diff 算法对比新旧 VNode 树，生成补丁；
6. 把补丁应用到真实 DOM，页面更新。

## 四、总结：每个环节的核心价值

|环节|核心作用|关键优化（Vue 3）|
|:--|:--|:--|
|**模板**|声明式描述页面，开发者友好|-|
|**AST**|把模板字符串转换成结构化 JS 对象，方便后续处理|-|
|**静态标记**|识别静态 / 动态节点，给动态节点打 PatchFlags|靶向更新，减少 Diff 开销|
|**render 函数**|把 AST 转换成可执行的 JS 代码|静态提升，避免重复创建节点|
|**虚拟 DOM**|用 JS 对象模拟真实 DOM，跨平台 + 性能优化|基于 PatchFlags 的 Diff 算法|
|**真实 DOM 更新**|把最小变化应用到页面|批量更新，减少重排重绘|