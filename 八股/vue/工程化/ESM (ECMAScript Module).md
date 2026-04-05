ESM（全称 ECMAScript Module，也叫 ES 模块）是 **ES2015（ES6）正式纳入 JavaScript 语言规范的官方模块化标准**，为 JavaScript 提供了原生、统一、跨环境的模块化能力，彻底解决了早期社区模块化方案（CommonJS、AMD、CMD）的碎片化问题，目前已成为浏览器、Node.js、前端工程化体系的通用模块化标准。

## 一、ESM 核心设计理念与基础特性

ESM 从语言层面定义了模块化规则，核心特性决定了它和传统模块化方案的本质区别，也是其成为主流标准的核心原因。

### 1. 静态化设计（最核心特性）

ESM 采用**编译时静态加载**机制：模块的导入（`import`）和导出（`export`）语句必须写在代码顶层，不能嵌套在条件判断、函数、循环中。JavaScript 引擎会在代码执行前，就完成模块依赖的解析和关联，而非运行时动态加载。

这个特性带来了两个关键优势：

- 支持 **Tree-Shaking（摇树优化）**：构建工具可以通过静态依赖分析，精准剔除未被使用的代码，大幅减小产物体积；
- 更快的加载速度：引擎可以提前并行下载、解析依赖模块，无需等待代码执行。

### 2. 模块级私有作用域

每个 ESM 模块都是一个独立的私有作用域，模块内顶层声明的变量、函数、类，仅在模块内部可访问，不会泄露到全局作用域，彻底解决了全局变量污染问题。只有通过 `export` 显式导出的内容，才能被外部模块访问。

### 3. 单例模式

同一个模块被多次 `import` 时，**仅会在第一次导入时执行一次模块代码**，后续导入都会复用同一个导出实例，不会重复执行模块代码，也不会生成多个导出副本。

### 4. 默认开启严格模式

所有 ESM 模块**默认启用严格模式（`use strict`）**，无需手动声明。这意味着模块内会强制遵循严格模式规则：

- 禁止使用未声明的变量；
- 顶层 `this` 指向 `undefined`（而非浏览器的 `window` 或 Node.js 的 `global`）；
- 禁止修改只读属性、删除不可配置属性等不安全操作。

### 5. 原生异步加载支持

浏览器环境中，ESM 模块默认采用异步加载机制，不会阻塞 HTML 文档的解析，行为等同于给普通脚本添加了 `defer` 属性，会在文档解析完成后、`DOMContentLoaded` 事件触发前按顺序执行。

## 二、ESM 核心语法

ESM 的语法分为两大部分：`export`（模块导出）和 `import`（模块导入），同时支持动态导入、重新导出等高级用法。

### 1. 模块导出（export）

`export` 用于将模块内的变量、函数、类等内容暴露给外部模块使用，分为 4 种常用形式。

#### （1）命名导出（Named Exports）

一个模块可以有多个命名导出，导出的内容必须有明确的名称，导入时需严格匹配名称。适合导出多个功能、工具函数的场景。

```javascript
// math.js 模块
// 单个声明时直接导出
export const PI = 3.1415926;
export function add(a, b) {
  return a + b;
}

// 批量统一导出（推荐，便于维护）
const minus = (a, b) => a - b;
function multiply(a, b) {
  return a * b;
}
export { minus, multiply };

// 导出时重命名
export { multiply as times };
```

#### （2）默认导出（Default Export）

每个模块**只能有一个默认导出**，适合模块仅对外提供一个核心功能的场景（比如一个类、一个主函数），导入时可以自定义名称，无需匹配原名称。

```javascript
// user.js 模块
// 导出函数
export default function sayHello(name) {
  return `Hello, ${name}`;
}

// 也可以导出类、对象、原始值
// export default class User { constructor(name) { this.name = name; } }
// export default { version: '1.0.0' };
```

#### （3）重新导出（Re-export）

将其他模块的内容直接从当前模块导出，常用于统一入口文件（比如 `index.js` 汇总整个目录的模块），简化外部导入路径。

```javascript
// index.js 统一入口
// 重新导出其他模块的命名导出
export { add, minus } from './math.js';
// 重新导出并重命名
export { multiply as calcMultiply } from './math.js';
// 重新导出其他模块的所有命名导出（不包含默认导出）
export * from './math.js';
// 重新导出其他模块的默认导出，转为命名导出
export { default as User } from './user.js';
```

#### （4）副作用导出

仅执行模块代码，不导出任何内容，常用于全局初始化、polyfill 注入、全局样式注册等场景。

```javascript
// init.js 模块
console.log('模块初始化执行');
window.globalConfig = { version: '1.0.0' };

// 其他模块导入，仅执行代码，不接收导出
import './init.js';
```

### 2. 模块导入（import）

`import` 用于从其他模块导入已导出的内容，与导出形式一一对应。

#### （1）命名导入

对应命名导出，必须用大括号包裹，名称必须和导出名称完全一致，支持重命名解决命名冲突。

```javascript
// 基础命名导入
import { add, minus, PI } from './math.js';
console.log(add(1, 2)); // 3
console.log(PI); // 3.1415926

// 导入时重命名
import { multiply as calcMultiply } from './math.js';
console.log(calcMultiply(2, 3)); // 6
```

#### （2）默认导入

对应默认导出，无需大括号，可以自定义导入名称。

```javascript
// 自定义名称导入默认导出
import sayHi from './user.js';
console.log(sayHi('ESM')); // Hello, ESM

// 命名导出 + 默认导出混合导入
import sayHi, { PI } from './user.js';
```

#### （3）整体命名空间导入

将模块所有导出的内容，挂载到一个命名空间对象上，避免命名污染，适合导入大量导出内容的场景。

```javascript
import * as MathModule from './math.js';
console.log(MathModule.add(1, 2)); // 3
console.log(MathModule.PI); // 3.1415926
```

### 3. 动态导入（Dynamic Import）

ES2020 新增的特性，解决了静态 `import` 无法动态加载、条件加载的问题。

动态导入通过 `import()` 函数实现，**可以在代码任意位置调用**，支持动态路径、条件加载，返回一个 Promise 对象，加载完成后会 resolve 模块的导出对象。

```javascript
// 1. 条件按需加载
if (needCalc) {
  import('./math.js').then((MathModule) => {
    MathModule.add(1, 2);
  }).catch(err => console.error('模块加载失败', err));
}

// 2. async/await 写法（推荐）
async function loadMathModule() {
  const { add, minus } = await import('./math.js');
  add(1, 2);
}

// 3. 动态路径加载
const lang = 'zh-CN';
import(`./locales/${lang}.js`).then((module) => {
  console.log(module.i18n);
});
```

**典型应用场景**：路由懒加载、按需加载非核心功能、根据用户环境加载不同模块。

## 三、ESM 核心运行机制

### 1. 模块加载三阶段

ESM 的加载过程分为三个串行阶段，和 CommonJS 的运行时加载有本质区别：

1. **构建 / 解析阶段**：下载模块文件，解析模块代码，识别 `import/export` 语句，构建模块依赖树，确定所有模块的依赖关系（此阶段不执行代码）。
2. **实例化阶段**：在内存中开辟空间，为模块 `export` 导出的变量创建引用（此时变量还未赋值），同时将导入语句和对应导出的变量建立关联，也就是**活绑定**。
3. **求值阶段**：从上到下执行模块代码，给 `export` 导出的变量完成赋值。

### 2. 活绑定（Live Binding）

活绑定是 ESM 和 CommonJS 最核心的区别之一：

- **CommonJS**：导出的是**值的拷贝 / 快照**，模块内的原始值变化后，导入处的取值不会同步更新；
- **ESM**：导出的是**变量的内存引用**，模块内的原始值变化后，导入处的取值会同步更新。

示例：

```javascript
// counter.js 导出模块
export let count = 0;
export function increment() {
  count++;
}

// main.js 导入模块
import { count, increment } from './counter.js';
console.log(count); // 0
increment();
console.log(count); // 1（同步更新，CommonJS 此处仍为 0）
```

## 四、ESM 跨环境支持

### 1. 浏览器环境

现代浏览器（Chrome 61+、Firefox 60+、Safari 10.1+、Edge 16+）均已原生支持 ESM，只需给 `script` 标签添加 `type="module"` 属性即可。

```html
<!-- 引入 ESM 模块 -->
<script type="module" src="./main.js"></script>

<!-- 内联 ESM 模块 -->
<script type="module">
  import { add } from './math.js';
  console.log(add(1, 2));
</script>
```

#### 浏览器环境关键规则

- 模块路径必须是**相对路径（`./`/`../`）、绝对路径或完整 URL**，不支持裸模块名（如 `import 'vue'`），可通过 `importmap` 解决；
- 模块遵循 CORS 跨域规则，跨域加载模块需要服务端配置正确的 CORS 响应头；
- 模块脚本默认 `defer` 行为，不会阻塞 HTML 解析，添加 `async` 属性可改为加载完成后立即执行；
- 可通过 `nomodule` 属性为不支持 ESM 的旧浏览器提供降级方案：
    
```html
    <script type="module" src="./main.js"></script>
    <script nomodule src="./main-legacy.js"></script>
```
#### Import Maps

浏览器原生提供的模块映射能力，解决裸模块名的导入问题，将模块名映射到对应的 URL 路径：

```html
<script type="importmap">
{
  "imports": {
    "vue": "https://unpkg.com/vue@3/dist/vue.esm-browser.js",
    "@/utils": "./src/utils/index.js"
  }
}
</script>

<script type="module">
  import { createApp } from 'vue';
  import { formatDate } from '@/utils';
</script>
```

### 2. Node.js 环境

Node.js 从 v12 开始稳定支持 ESM，v14+ 完全支持，目前已成为 Node.js 推荐的模块化方案。
#### 启用方式

1. **package.json 配置**：在 `package.json` 中设置 `"type": "module"`，此时目录下所有 `.js` 文件都会被当作 ESM 模块处理；
2. **后缀名指定**：将文件后缀改为 `.mjs`，无论 package.json 如何配置，都会被当作 ESM 模块处理；
3. 若需在 ESM 项目中使用 CommonJS 模块，需将文件后缀改为 `.cjs`。
#### Node.js ESM 关键注意事项

- ESM 中没有 CommonJS 的内置变量：`__dirname`、`__filename`、`require`、`module`、`exports`，需手动实现：
    
```javascript
    import { fileURLToPath } from 'url';
    import { dirname } from 'path';
    const __filename = fileURLToPath(import.meta.url);
    const __dirname = dirname(__filename);
```

- 支持导入 CommonJS 模块，但仅支持默认导入（CommonJS 的 `module.exports` 会被当作 `default` 导出），不支持命名导入；
- 原生 JSON 导入需添加导入断言：`import data from './data.json' assert { type: 'json' };`。

## 五、ESM 高级特性

### 1. 顶层 await（Top-level await）

ES2022 正式纳入规范，允许在模块顶层直接使用 `await`，无需包裹在 `async` 函数中。模块会等待 `await` 执行完成后，再完成实例化和求值，同时会等待依赖模块的顶层 await 完成。

```javascript
// config.js 模块
export const config = await fetch('/api/config').then(res => res.json());

// main.js 模块
import { config } from './config.js';
// 此处会等待 config.js 的 await 完成，config 已完成赋值
console.log(config);
```

**适用场景**：模块初始化配置加载、异步资源初始化、条件加载模块。

### 2. import.meta

模块内置的元信息对象，提供当前模块的上下文信息，浏览器和 Node.js 均支持，最常用的是 `import.meta.url`（当前模块的绝对 URL 路径）。

```javascript
// 浏览器/Node.js 通用：获取当前模块的URL
console.log(import.meta.url);

// 浏览器环境：获取模块所在的基础路径
const baseUrl = new URL('./', import.meta.url).href;

// Node.js 环境：自定义元信息
// 启动命令：node --experimental-import-meta-resolve main.js
console.log(import.meta.resolve('./math.js'));
```

### 3. 导入断言（Import Assertions）

ES2022 新增特性，用于显式指定导入文件的类型，防止 MIME 类型嗅探攻击，确保服务端返回的文件类型符合预期，目前主要用于 JSON、CSS 等非 JS 模块的导入。

```javascript
// 导入 JSON 模块
import data from './data.json' assert { type: 'json' };

// 浏览器原生导入 CSS 模块
import styles from './main.css' assert { type: 'css' };
document.adoptedStyleSheets.push(styles);
```

## 六、ESM 与 CommonJS 核心区别对比

|对比维度|ESM（ES 模块）|CommonJS（CJS）|
|:--|:--|:--|
|标准归属|ECMAScript 官方语言标准，原生支持|Node.js 社区规范，非语言原生|
|加载机制|静态加载，编译时解析依赖关系|动态加载，运行时执行到 `require` 才加载|
|导出机制|活绑定（引用传递），原值变化同步更新|值拷贝（快照），原值变化导入处不更新|
|作用域|模块级私有作用域，默认开启严格模式|函数级作用域，默认非严格模式|
|顶层 this|指向 `undefined`|指向 `module.exports`|
|异步能力|原生支持动态 `import()`、顶层 await|`require` 同步加载，不支持顶层 await|
|跨环境|浏览器、Node.js 全环境通用|仅原生支持 Node.js，浏览器需构建工具转换|
|Tree-Shaking|原生支持（静态依赖分析）|不支持（动态加载无法静态分析）|
## 七、ESM 最佳实践

1. **优先使用命名导出**：命名导出更利于 Tree-Shaking 优化，也便于代码提示和重构，默认导出仅用于模块只有一个核心功能的场景。
2. **静态导入统一放在文件顶部**：符合静态化设计原则，便于代码阅读和依赖分析，动态导入仅用于按需加载、条件加载场景。
3. **避免循环依赖**：ESM 虽支持循环依赖，但容易出现变量未初始化的问题，应通过公共模块抽离、代码分层避免循环依赖。
4. **统一模块化规范**：项目中尽量避免 ESM 和 CommonJS 混合使用，减少兼容性问题。
5. **合理使用顶层 await**：避免滥用顶层 await 阻塞依赖模块的加载，仅用于模块必须的初始化操作。