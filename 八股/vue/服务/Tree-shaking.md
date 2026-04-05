Tree-shaking（摇树优化）是前端工程化中**基于 ES6 模块（ESM）静态特性的死代码消除（DCE）技术**，核心作用是在打包阶段自动剔除项目中未被引用、未被执行的冗余代码，大幅减小最终打包产物的体积，提升页面加载性能。

术语源自「摇树」的直观比喻：把项目的依赖关系看作一棵代码树，打包工具像摇动树干一样，只保留被实际引用、执行的「活代码」（绿叶），剔除从未被使用的「死代码」（枯叶）。它最早由 Rollup 实现，目前 Webpack、Vite 等主流打包工具均已原生支持，是前端性能优化的必备手段。

## 一、核心实现原理

Tree-shaking 的底层依赖两大核心能力：**ES6 模块的静态可分析性**，以及**打包工具 + 压缩工具的协同标记与消除**。

### 1. 基础前提：ES6 模块的静态特性

Tree-shaking 能实现的根本原因，是 ES6 模块（ESM）的**静态语法约束**：

- `import`/`export` 语句必须写在模块顶层，不能嵌套在函数、条件语句中；
- 模块导入路径必须是静态字符串常量，不能是运行时计算的动态表达式；
- 导入导出关系在**编译阶段**就能完全确定，无需等待代码运行。

与之相对，CommonJS 的 `require()` 是动态函数调用，路径可以是运行时计算的变量、可以写在条件分支里，打包工具在不执行代码的情况下，无法提前确定模块的依赖和使用情况，因此**几乎无法对 CommonJS 模块做有效的 Tree-shaking**。

### 2. 核心执行流程：标记 + 消除

Tree-shaking 分为两个核心阶段，由打包工具和压缩工具协同完成，二者缺一不可：

1. **标记阶段（打包工具负责）**
    
    从项目入口文件开始，解析代码生成抽象语法树（AST），递归追踪所有 `import` 语句，构建完整的模块依赖图；顺着依赖图，精准标记出所有被实际引用的导出成员、以及从未被使用的「死代码」。
2. **消除阶段（压缩工具负责）**
    
    打包工具仅做标记，不会直接删除代码；真正的死代码剔除，由 Terser、UglifyJS 等压缩工具在生产环境压缩时完成 —— 压缩工具会根据打包工具的标记，安全删除未被引用的代码，同时完成代码混淆、压缩等操作。

## 二、Tree-shaking 生效的四大必要条件

项目中 Tree-shaking 不生效，99% 的情况是未满足以下全部条件，必须同时满足才能实现完整的摇树优化：

|必要条件|详细要求|
|---|---|
|模块语法|必须全程使用 **ES6 Module（import/export）**，禁止代码被转译为 CommonJS 格式|
|运行环境|必须开启 **production 生产模式**，开发环境为了保留调试信息，不会执行死代码消除|
|压缩配合|必须开启代码压缩，且使用支持 DCE 的压缩工具（Webpack5 内置 Terser，默认开启）|
|副作用声明|必须通过 `package.json` 的 `sideEffects` 字段，明确标记代码的副作用，避免打包工具因安全顾虑保留冗余代码|

### 关键概念：什么是副作用？

副作用，指代码执行时，除了导出成员之外，还会对外部环境产生可观测的影响。典型的副作用包括：

- 修改全局变量、window 对象、原生原型链；
- 顶层执行的 `console.log`、网络请求、定时器；
- 立即执行函数表达式（IIFE）；
- CSS、全局 polyfill 等导入即生效的文件。

打包工具的核心原则是：**只要模块存在未声明的副作用，哪怕其导出成员未被使用，也不敢随意删除**，避免影响项目正常运行。因此必须通过 `package.json` 的 `sideEffects` 字段明确声明：

```json
{
  "name": "your-project",
  // 方式1：整个项目所有文件均无副作用，可安全摇树
  "sideEffects": false,
  // 方式2：仅标记指定文件有副作用，其余文件无副作用
  "sideEffects": ["*.css", "*.scss", "./src/polyfill.js"]
}
```

## 三、Tree-shaking 与传统 DCE 的区别

很多人会混淆 Tree-shaking 和 DCE（死代码消除），二者核心目标一致，但实现逻辑、作用范围完全不同，Tree-shaking 是 DCE 的一种专项实现，二者是互补而非替代关系。

|特性|Tree-shaking|传统 DCE（死代码消除）|
|---|---|---|
|核心逻辑|包含法：从入口出发，只保留被明确引用的「活代码」|排除法：全量接收代码，只删除明确不可达的「死代码」|
|分析依据|基于 ESM 模块的静态依赖关系，跨模块全局分析|基于代码执行逻辑，单文件 / 单作用域内分析|
|作用范围|模块级、导出成员级，消除跨模块未被引用的代码|代码块级，消除不可达的代码（如 `if(false)` 内的代码、return 后的代码）|
|执行阶段|打包的模块依赖解析阶段|代码压缩阶段|
|依赖前提|必须使用 ESM 静态模块，依赖副作用声明|无模块格式要求，只要能确定代码不可达即可执行|

举个直观的例子：

```javascript
// 传统DCE能消除的代码：永远不会执行的代码块
function test() {
  const a = 1;
  return a;
  const b = 2; // return 后的代码，DCE会自动删除
  console.log(b);
}

// Tree-shaking能消除的代码：导出但未被引用的成员
// math.js
export const add = (a, b) => a + b; // 被引用，保留
export const minus = (a, b) => a - b; // 导出但未被引用，Tree-shaking删除
export const multiply = (a, b) => a * b; // 导出但未被引用，Tree-shaking删除

// 入口文件
import { add } from './math.js';
console.log(add(1, 2));
```

## 四、高频失效场景与避坑指南

实际开发中，绝大多数 Tree-shaking 失效问题，都来自以下场景，可直接对照排查：

### 1. ESM 被转译为 CommonJS

这是最常见的失效原因。Babel/TypeScript 编译时，默认会把 ESM 的 `import/export` 转成 CommonJS 的 `require/module.exports`，导致静态依赖关系丢失，Tree-shaking 完全失效。

- 修复方案（Babel）：`@babel/preset-env` 配置 `modules: false`，禁止转译模块语法
    
```javascript
    // babel.config.js
    module.exports = {
      presets: [
        ['@babel/preset-env', {
          modules: false, // 核心：保留ESM模块，不转成CommonJS
          useBuiltIns: 'usage',
          corejs: 3
        }]
      ]
    }
```
    
- 修复方案（TypeScript）：`tsconfig.json` 配置 `module: ESNext/ES6`
    
```json
    {
      "compilerOptions": {
        "module": "ESNext", // 输出ESM模块
        "moduleResolution": "NodeNext"
      }
    }
```

### 2. 副作用标记错误 / 缺失

- 未配置 `sideEffects` 字段：打包工具会保守地认为所有文件都有副作用，整包保留，不会执行摇树；
- 误标记有副作用的文件：比如 CSS 文件未被标记，会被打包工具当成无副作用的代码删除，导致样式丢失；
- 代码中存在隐式副作用：比如顶层修改全局变量、执行网络请求，哪怕配置了 `sideEffects: false`，打包工具也会保留整个模块。

### 3. 导出方式错误

- 禁止默认导出一个大对象：打包工具无法静态分析对象的属性使用情况，哪怕只用了一个方法，整个对象都会被保留
    
```javascript
    // ❌ 错误写法：无法Tree-shaking
    export default {
      add: (a, b) => a + b,
      minus: (a, b) => a - b
    }
    
    // ✅ 正确写法：粒度化具名导出
    export const add = (a, b) => a + b;
    export const minus = (a, b) => a - b;
```
    
- 禁止通过中间变量间接导出：会破坏静态分析链路，导致摇树失效。

### 4. 第三方库不兼容

- 很多第三方库默认只提供 CommonJS 版本（如旧版 lodash），全量导入时 Tree-shaking 完全失效；
- 修复方案：切换到库的 ESM 版本，比如用 `lodash-es` 替代 `lodash`，用 `antd/es` 替代 `antd/lib`；
- 禁止全量导入库：比如 `import _ from 'lodash'` 会把整个库打包进去，必须用按需导入 `import debounce from 'lodash-es/debounce'`。

### 5. 动态导入 / 动态引用

动态路径的导入、条件导出，会让打包工具无法静态分析依赖，只能把所有可能的模块全部打包，导致摇树失效。

```javascript
// ❌ 错误写法：动态路径，无法静态分析
const module = import(`./utils/${name}.js`);

// ❌ 错误写法：条件导出，无法静态分析
if (process.env.NODE_ENV === 'development') {
  export const debug = () => console.log('debug');
}
```

### 6. 其他常见误区

- 在开发环境测试 Tree-shaking：开发环境为了保留调试信息，只会标记未使用代码，不会真正删除，必须在生产环境构建后验证；
- 滥用闭包和函数包装：会让压缩工具无法确定代码是否有副作用，不敢安全删除；
- 未开启压缩工具：打包工具只做标记，没有 Terser 等压缩工具，死代码永远不会被删除。

## 五、主流打包工具的支持情况

|打包工具|支持程度|核心特点|
|---|---|---|
|Rollup|原生完美支持|最早实现 Tree-shaking，天生基于 ESM 设计，摇树精度最高、效果最好，是前端库打包的首选|
|Vite|原生完美支持|开发环境用 Rollup 做预构建，生产环境直接用 Rollup 打包，完整继承 Rollup 的 Tree-shaking 能力，默认开启，无需额外配置|
|Webpack|完善支持|Webpack 2 开始支持，Webpack 4+ 已非常成熟，生产模式默认开启；Webpack 5 新增深层摇树优化，支持对嵌套对象的属性做精准摇树|

#### Webpack 完整配置示例

```javascript
// webpack.config.js
module.exports = {
  mode: 'production', // 核心：生产模式自动开启基础优化
  entry: './src/index.js',
  output: {
    filename: 'bundle.js',
    clean: true
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: 'babel-loader' // babel需配置保留ESM模块
      }
    ]
  },
  optimization: {
    usedExports: true, // 开启未使用导出标记，生产模式默认开启
    minimize: true, // 开启压缩，生产模式默认开启
    // 内置Terser压缩工具，可自定义配置
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            pure_funcs: ['console.log'], // 标记无副作用的函数，可安全删除
            drop_debugger: true
          }
        }
      })
    ]
  }
}
```

## 六、进阶优化技巧：最大化 Tree-shaking 效果

1. **优先使用具名导出，避免默认导出大对象**：粒度化的具名导出，能让打包工具精准标记每个成员的使用情况，摇树效果最好。
2. **精准配置 `sideEffects` 字段**：只给真正有副作用的文件做标记，其余文件全部声明为无副作用，最大化摇树空间。
3. **全链路使用 ESM 规范**：项目源码、第三方依赖、编译配置，全程统一使用 ESM，禁止 CommonJS 混入。
4. **使用 `/*#__PURE__*/` 标记纯函数**：该注释告诉压缩工具，这个函数调用无任何副作用，如果返回值未被使用，可安全删除。
    
```javascript
    // 标记该函数调用是纯的，未被使用时可直接删除
    const utils = /*#__PURE__*/ createUtils();
    export const format = /*#__PURE__*/ (str) => str.trim();
```
    
5. **避免顶层副作用代码**：不要在模块顶层执行 `console.log`、网络请求、修改全局变量等操作，把副作用逻辑放到函数内部执行。
6. **拆分大模块**：把一个大模块拆分成多个小的、功能单一的子模块，避免单个模块内的副作用影响整个模块的摇树效果。

## 七、验证 Tree-shaking 是否生效

1. 执行生产环境构建命令（如 `npm run build`）；
2. 打开打包后的产物文件，搜索未被使用的导出成员名称，如果完全找不到，说明摇树生效；
3. 借助工具验证：
    - Webpack：使用 `webpack-bundle-analyzer` 插件，可视化查看打包产物的组成，定位未被摇掉的冗余代码；
    - Rollup/Vite：使用 `rollup-plugin-visualizer` 插件，做同样的产物分析。