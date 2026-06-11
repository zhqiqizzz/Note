
# Vite 的完整流程

我们专门把 **Vite 的流程**讲细一点。你可以把 Vite 理解成两套流程：

```
开发环境：dev server，核心是“不打包 + 原生 ESM + 按需转换”
生产环境：build，核心是“Rollup 打包 + 压缩优化 + 生成静态资源”
```

这是 Vite 和 webpack 最大的思路差异。

---

**一、Vite 为什么开发环境快**

传统 webpack dev server 通常是：

```
启动项目
-> 从入口文件开始分析依赖
-> 把很多模块打包到内存里
-> 浏览器加载 bundle
-> 页面运行
```

也就是说，项目启动时要先做一次比较完整的打包。

项目越大，启动越慢。

Vite 开发环境是：

```
启动项目
-> 启动一个开发服务器
-> 不提前打包你的业务代码
-> 浏览器请求哪个模块，Vite 才处理哪个模块
```

也就是：

> 按需编译，而不是启动时全量打包。

---

**二、Vite 开发环境整体流程**

假设你执行：

```
npm run dev
```

通常对应：

```
{
  "scripts": {
    "dev": "vite"
  }
}
```

Vite 会启动一个本地服务器，例如：

```
http://localhost:5173
```

然后浏览器访问页面。

整体流程是：

```
1. Vite 启动 dev server
2. 浏览器请求 index.html
3. Vite 把 index.html 当作入口处理
4. 浏览器加载 <script type="module" src="/src/main.js">
5. 浏览器按 ESM 规则继续请求 main.js 里的 import
6. Vite 拦截这些模块请求
7. 如果是 .vue / .ts / css / 图片 / node_modules，Vite 转换后返回
8. 浏览器执行模块，页面渲染
```

注意一个关键点：

> 在 Vite 里，`index.html` 是入口的一部分，不只是静态文件。

---

**三、浏览器请求 index.html**

你的项目可能是这样：

```
<div id="app"></div>

<script type="module" src="/src/main.js"></script>
```

浏览器请求：

```
GET /
```

Vite 返回 `index.html`。

然后浏览器看到：

```
<script type="module" src="/src/main.js"></script>
```

于是请求：

```
GET /src/main.js
```

---

**四、浏览器请求 main.js**

`main.js`：

```
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App).use(router).mount('#app')
```

浏览器会继续发请求：

```
GET /src/App.vue
GET /src/router/index.js
GET /node_modules/vue/...
```

但问题来了，浏览器并不能直接理解：

```
import { createApp } from 'vue'
```

因为浏览器不知道 `vue` 这个裸模块名要去哪找。

浏览器也不能直接执行：

```
import App from './App.vue'
```

因为浏览器不认识 `.vue` 文件。

所以 Vite dev server 会介入。

---

**五、Vite 会重写 import**

开发环境下，Vite 会把：

```
import { createApp } from 'vue'
```

转换成类似：

```
import { createApp } from '/node_modules/.vite/deps/vue.js?v=xxxxx'
```

也就是把裸模块名 `vue` 改写成浏览器能请求的路径。

这一步叫：

```
模块解析 / import 重写
```

为什么要重写？

因为浏览器只能识别这些路径：

```
import './a.js'
import '../utils/index.js'
import '/src/main.js'
import 'https://cdn.xxx.com/vue.js'
```

它不能直接识别：

```
import vue from 'vue'
```

这是构建工具要帮你做的事。

---

**六、Vite 怎么处理 .vue 文件**

比如：

```
import App from './App.vue'
```

浏览器请求：

```
GET /src/App.vue
```

Vite 收到这个请求后，会用 Vue 插件处理它。

一个 `.vue` 文件大概是这样：

```
<template>
  <div>{{ message }}</div>
</template>

<script setup>
const message = 'hello vite'
</script>

<style scoped>
div {
  color: red;
}
</style>
```

浏览器不能直接运行这个文件。

Vite 会把它拆成几部分处理：

```
template -> 编译成 render 函数
script -> 转成 JS 模块
style -> 转成 CSS 并通过 JS 注入页面
```

最后返回给浏览器的是可以执行的 JS 模块。

大概类似：

```
const message = 'hello vite'

function render() {
  // 编译后的渲染函数
}

export default {
  setup() {
    return { message }
  },
  render
}
```

真实代码会更复杂，但你理解这个意思就够了。

---

**七、Vite 怎么处理 CSS**

如果你在 `main.js` 写：

```
import './style.css'
```

浏览器原生 JS 不支持直接 import CSS。

Vite 开发环境会把 CSS 转成一个 JS 模块。

它大概会做这样的事：

```
const css = `
body {
  margin: 0;
}
`

const style = document.createElement('style')
style.textContent = css
document.head.appendChild(style)
```

所以开发环境里，CSS 是通过 JS 动态插入到页面中的。

这也方便热更新。

当你改了 CSS，Vite 不需要刷新整个页面，只需要替换对应的 `<style>` 内容。

---

**八、Vite 的依赖预构建**

这是 Vite 非常关键的一步，叫：

```
Dependency Pre-Bundling
依赖预构建
```

Vite 开发环境虽然业务代码不打包，但它会对 `node_modules` 里的依赖做预构建。

比如：

```
import { debounce } from 'lodash-es'
import axios from 'axios'
import { createApp } from 'vue'
```

Vite 会用 esbuild 提前处理这些依赖，生成到：

```
node_modules/.vite/deps
```

为什么要预构建？

主要有两个原因。

---

**原因 1：把 CommonJS 转成 ESM**

有些 npm 包是 CommonJS 格式：

```
const axios = require('axios')
module.exports = axios
```

但浏览器原生 ESM 不认识 CommonJS。

所以 Vite 要把这些依赖转成浏览器能运行的 ESM。

---

**原因 2：减少浏览器请求数量**

有些库内部文件很多。

比如你导入：

```
import _ from 'lodash-es'
```

它内部可能有很多小模块。

如果完全交给浏览器按 ESM 一个个请求，可能会产生大量网络请求。

Vite 会用 esbuild 把依赖预构建成少量文件，比如：

```
/node_modules/.vite/deps/lodash-es.js
```

这样开发环境更快。

---

**九、为什么是 esbuild**

Vite 开发环境很多转换任务用的是 esbuild。

esbuild 是用 Go 写的，速度非常快。

Vite 用它做：

- 依赖预构建
- TypeScript 转 JS
- JSX 转 JS
- 部分语法转换

所以 Vite 开发体验很快。

但注意：

> Vite 生产构建默认不是用 esbuild 打包，而是用 Rollup 打包。

esbuild 更像是开发阶段的高速转换器和压缩工具之一。

---

**十、Vite 的热更新 HMR**

HMR 全称：

```
Hot Module Replacement
```

热模块替换。

意思是：

> 修改某个模块后，不刷新整个页面，只替换变化的模块。

比如你改了一个 Vue 组件：

```
<template>
  <div>Hello</div>
</template>
```

Vite 会做：

```
1. 监听文件变化
2. 找到变化的模块
3. 通过 WebSocket 通知浏览器
4. 浏览器请求新的模块代码
5. 只替换对应模块
6. 尽量保留页面状态
```

比如表单里已经输入了内容，如果只是改组件样式，页面不会整页刷新，状态可以保留。

开发时你会看到浏览器和 Vite dev server 之间有一个 WebSocket 连接，就是用来推送 HMR 消息的。

---

**十一、Vite 生产构建流程**

当你执行：

```
npm run build
```

通常对应：

```
{
  "scripts": {
    "build": "vite build"
  }
}
```

这时流程和开发环境不一样。

生产环境不会让浏览器一个模块一个模块请求源码，而是会打包。

流程大概是：

```
1. 读取 vite.config.js
2. 以 index.html 作为入口
3. 找到 script module 入口，比如 /src/main.js
4. Rollup 从入口开始构建依赖图
5. 处理 .vue、.ts、css、图片等资源
6. Tree Shaking 删除未使用代码
7. 代码分包
8. 压缩 JS / CSS
9. 给文件名加 hash
10. 输出 dist 目录
```

输出可能是：

```
dist/
  index.html
  assets/
    index-a8d3f2.js
    vendor-9f13ab.js
    index-7cd91.css
    logo-2f8a1.png
```

---

**十二、为什么生产环境要 hash**

比如构建后生成：

```
index-a8d3f2.js
```

如果代码变了，下次可能生成：

```
index-b91caa.js
```

这样浏览器就知道这是一个新文件，会重新请求。

如果文件名一直叫：

```
index.js
```

用户浏览器可能会使用旧缓存，导致上线后用户还看到旧代码。

所以 hash 是为了配合缓存。

---

**十三、开发环境和生产环境区别**

|场景|开发环境|生产环境|
|---|---|---|
|命令|`vite`|`vite build`|
|核心目标|启动快、更新快|体积小、加载快、可缓存|
|是否打包业务代码|通常不打包|打包|
|模块加载|浏览器原生 ESM 按需请求|加载构建后的静态资源|
|主要工具|esbuild + dev server|Rollup + esbuild|
|CSS|JS 注入 style|提取成 CSS 文件|
|HMR|支持|不需要|
|文件名 hash|不需要|需要|

---

**十四、用一段代码串起来**

开发时你写：

```
import { createApp } from 'vue'
import App from './App.vue'
import './style.css'

createApp(App).mount('#app')
```

Vite 开发环境做的事：

```
1. 把 vue 解析成 /node_modules/.vite/deps/vue.js
2. 把 App.vue 编译成 JS 模块
3. 把 style.css 转成 JS，动态插入 style 标签
4. 浏览器按 ESM 一个个请求这些模块
5. 文件变化时通过 WebSocket 做 HMR
```

生产构建做的事：

```
1. 从 main.js 出发分析整个依赖图
2. 编译 App.vue
3. 处理 CSS，并提取成单独 CSS 文件
4. Tree Shaking 删除没用代码
5. 分包、压缩、生成 hash 文件名
6. 输出 dist
```

---

**十五、面试版总结**

你可以这样答：

> Vite 分为开发环境和生产环境两套流程。开发环境下，Vite 启动 dev server，以 `index.html` 作为入口，利用浏览器原生 ESM，不会一开始就把业务代码全部打包，而是浏览器请求哪个模块，Vite 就按需转换哪个模块。对于浏览器不能直接识别的内容，比如 `.vue`、TypeScript、CSS、裸模块导入，Vite 会在 dev server 中进行编译和 import 重写。
> 
> 同时 Vite 会对 `node_modules` 做依赖预构建，用 esbuild 把 CommonJS 或模块数量很多的依赖提前转换成 ESM，放到 `node_modules/.vite/deps`，这样可以减少请求数量并提升启动速度。开发时文件变化后，Vite 通过 WebSocket 通知浏览器进行 HMR，实现快速更新。
> 
> 生产环境下，Vite 会使用 Rollup 从入口构建完整依赖图，进行 Tree Shaking、代码分包、CSS 提取、资源处理、压缩和文件名 hash，最终输出 `dist` 静态资源。所以 Vite 开发快是因为基于原生 ESM 和按需转换，生产构建则仍然会打包优化。

# Webpack 的完整流程

现在进入 Webpack。

Webpack 的核心思想是：

> 从入口文件开始，递归分析所有依赖，把各种资源处理成模块，最后打包输出浏览器能运行的文件。

---

**一、Webpack 项目入口**

配置一般长这样：

```
// webpack.config.js
const path = require('path')

module.exports = {
  mode: 'development',

  entry: './src/main.js',

  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js'
  }
}
```

入口：

```
entry: './src/main.js'
```

意思是：

> Webpack 从 `src/main.js` 开始分析项目。

---

**二、从入口开始构建依赖图**

比如：

`main.js`：

```
import { createApp } from 'vue'
import App from './App.vue'
import './style.css'

createApp(App).mount('#app')
```

Webpack 会分析出：

```
main.js
  -> vue
  -> App.vue
  -> style.css
```

然后继续分析 `App.vue` 里面又依赖了什么：

```
<script>
import Hello from './components/Hello.vue'

export default {
  components: {
    Hello
  }
}
</script>
```

依赖图变成：

```
main.js
  -> vue
  -> App.vue
       -> Hello.vue
  -> style.css
```

Webpack 会递归做这个过程。

---

**三、Webpack 默认只认识 JS 和 JSON**

这是一个重点。

Webpack 默认能处理：

```
.js
.json
```

但不能直接处理：

```
.vue
.css
.scss
.png
.svg
.ts
```

所以需要 loader。

---

**四、loader 是什么**

loader 的作用是：

> 把 Webpack 不认识的文件，转换成 Webpack 能理解的模块。

比如 CSS：

```
import './style.css'
```

Webpack 默认不认识 CSS，所以要配置：

```
module: {
  rules: [
    {
      test: /\.css$/,
      use: ['style-loader', 'css-loader']
    }
  ]
}
```

这里执行顺序是从右到左：

```
css-loader -> style-loader
```

`css-loader` 做什么？

```
解析 CSS 文件，把它变成 JS 能 import 的模块
```

`style-loader` 做什么？

```
运行时创建 <style> 标签，把 CSS 插入页面
```

所以：

```
import './style.css'
```

最后会变成类似：

```
const style = document.createElement('style')
style.innerHTML = 'body { margin: 0 }'
document.head.appendChild(style)
```

---

**五、处理 Vue 文件**

如果是 Vue 项目，需要：

```
{
  test: /\.vue$/,
  loader: 'vue-loader'
}
```

`vue-loader` 会处理：

```
<template>
  <div>{{ msg }}</div>
</template>

<script>
export default {
  data() {
    return {
      msg: 'hello'
    }
  }
}
</script>

<style scoped>
div {
  color: red;
}
</style>
```

它会把 `.vue` 单文件组件拆开：

```
template -> 编译成 render 函数
script -> JS 模块
style -> 交给 css-loader / style-loader
```

最后让整个 `.vue` 文件变成 Webpack 能打包的 JS 模块。

---

**六、处理图片等静态资源**

Webpack 5 里可以用 Asset Modules：

```
module: {
  rules: [
    {
      test: /\.(png|jpg|gif|svg)$/,
      type: 'asset'
    }
  ]
}
```

Webpack 会根据资源大小决定：

- 小文件转成 base64
- 大文件输出成单独文件

比如你写：

```
import logo from './logo.png'
```

Webpack 处理后，`logo` 可能变成：

```
'/assets/logo.8df12.png'
```

然后你可以：

```
img.src = logo
```

---

**七、plugin 是什么**

loader 是处理单个模块文件的。

plugin 是参与 Webpack 构建流程的。

比如：

```
const HtmlWebpackPlugin = require('html-webpack-plugin')
const { VueLoaderPlugin } = require('vue-loader')

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html'
    }),
    new VueLoaderPlugin()
  ]
}
```

`HtmlWebpackPlugin` 做什么？

```
根据模板生成 dist/index.html
并自动把打包后的 JS / CSS 注入进去
```

比如输出：

```
<div id="app"></div>
<script src="bundle.js"></script>
```

`VueLoaderPlugin` 是配合 `vue-loader` 工作的。

一句话：

> loader 负责转换文件，plugin 负责扩展整个构建过程。

---

**八、Webpack 打包后产物是什么**

开发时你写很多模块：

```
src/main.js
src/App.vue
src/style.css
src/components/Hello.vue
node_modules/vue
```

Webpack 最后可能输出：

```
dist/
  index.html
  bundle.js
```

或者生产环境会有：

```
dist/
  index.html
  js/
    app.8d3f21.js
    vendor.91ab2c.js
  css/
    app.73ab9.css
  assets/
    logo.221fe.png
```

浏览器最后加载的是这些打包后的文件，而不是你的源码文件。

---

**九、Webpack 开发环境流程**

执行：

```
npm run serve
```

或者：

```
webpack serve
```

Webpack dev server 会做：

```
1. 读取 webpack 配置
2. 从 entry 开始构建依赖图
3. 使用 loader 转换各种模块
4. 使用 plugin 扩展构建流程
5. 把打包结果放到内存中
6. 启动本地服务器
7. 浏览器访问页面并加载 bundle
8. 文件变化后重新编译相关模块
9. 通过 HMR 或 live reload 更新页面
```

注意：

> Webpack dev server 通常不会真的把文件写入 dist，而是把构建结果放在内存中，提高开发速度。

---

**十、Webpack 生产构建流程**

执行：

```
npm run build
```

对应：

```
webpack --mode production
```

生产流程大概是：

```
1. 读取配置
2. 从 entry 开始递归分析依赖
3. loader 转换模块
4. plugin 参与优化和输出
5. Tree Shaking 删除没用代码
6. 代码压缩
7. CSS 提取
8. 代码分割
9. 文件名加 hash
10. 输出到 dist
```

生产环境更关注：

- 体积更小
- 加载更快
- 缓存更稳定

---

**十一、Webpack 的核心流程图**

```
entry
  ↓
递归分析依赖
  ↓
遇到不同资源，交给 loader 转换
  ↓
形成 dependency graph 依赖图
  ↓
plugin 参与构建生命周期
  ↓
chunk 分组
  ↓
生成 bundle
  ↓
输出 dist
```

这几个词很重要：

```
entry：入口
loader：模块转换
plugin：构建扩展
dependency graph：依赖图
chunk：代码块
bundle：最终打包文件
output：输出
```

---

**十二、Vite 和 Webpack 流程对比**

|对比项|Vite 开发环境|Webpack 开发环境|
|---|---|---|
|启动方式|启动 dev server，不全量打包业务代码|从 entry 开始打包|
|模块加载|浏览器原生 ESM 按需请求|浏览器加载 bundle|
|业务代码处理|请求到哪个模块处理哪个|启动时构建依赖图|
|依赖处理|node_modules 预构建|统一进入打包流程|
|HMR|基于 ESM 边界，更新快|基于 bundle 和模块热替换|
|生产构建|Rollup 打包|Webpack 打包|

一句话：

> Vite 开发时把一部分模块加载工作交给浏览器原生 ESM，所以启动快；Webpack 则是自己先构建依赖图并打包成 bundle，再交给浏览器运行。

---

**十三、面试版总结**

你可以这样说：

> ESM 是 JavaScript 官方模块规范，也就是 `import/export`。现代浏览器可以通过 `<script type="module">` 原生加载 ESM，浏览器会解析模块依赖并继续请求子模块。但浏览器只认识明确路径，不认识 `vue` 这种裸模块导入，也不认识 `.vue`、`.ts`、`.css` 这些非标准模块，所以仍然需要构建工具处理。
> 
> Webpack 的流程是从 `entry` 入口开始，递归分析模块依赖，构建 dependency graph。遇到 CSS、Vue、图片、TS 等资源时，使用 loader 把它们转换成 Webpack 能理解的模块；plugin 则参与整个构建生命周期，比如生成 HTML、提取 CSS、压缩代码。最后 Webpack 会把模块分组成 chunk，生成 bundle，并输出到 dist。开发环境下 webpack dev server 通常把 bundle 放在内存中并支持 HMR，生产环境会做 Tree Shaking、代码压缩、代码分割和文件名 hash。