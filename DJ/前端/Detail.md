
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

# 重排和重绘

**浏览器渲染大致流程**

页面从代码到屏幕，大概经历：

```
HTML -> DOM
CSS -> CSSOM
DOM + CSSOM -> Render Tree
计算布局 Layout/Reflow
绘制 Paint
合成 Composite
```

其中 **Layout/Reflow** 做的是：

> 计算每个元素在页面里的几何信息，比如宽、高、位置、边距、换行、滚动区域等。

比如浏览器要知道：

```
这个 div 宽多少？
这个 p 文字换几行？
这个元素距离视口顶部多少？
父元素会不会被子元素撑高？
```

这些都属于布局计算。

---

**什么是回流**

回流，也叫 Layout / Reflow，就是浏览器重新计算元素几何信息。

比如你改了：

```
box.style.width = '300px'
```

浏览器会知道：

> 这个元素宽度变了，可能影响自己、子元素、后面的兄弟元素、父元素高度。

于是相关区域的布局需要重新计算。

会触发回流的常见操作：

```
el.style.width = '100px'
el.style.height = '100px'
el.style.margin = '20px'
el.style.padding = '20px'
el.style.display = 'none'
el.classList.add('large')
document.body.appendChild(newNode)
```

这些都会改变布局。

---

**为什么读取 offsetHeight 会强制回流**

关键点是：浏览器为了性能，并不是你一改样式就立刻重新布局。

比如：

```
box.style.width = '300px'
box.style.height = '200px'
box.style.marginTop = '20px'
```

浏览器可能会先把这些修改标记为：

```
布局已脏，需要稍后重新计算
```

它不会每行 JS 都立刻重排，因为那样太慢。

正常情况下，浏览器会等 JS 执行完，在下一帧统一做布局和绘制。

但是如果你在修改样式后，马上读取布局信息：

```
box.style.width = '300px'

console.log(box.offsetHeight)
```

浏览器就必须回答你：

```
当前 box 的最新高度是多少？
```

可是前面的 `width = 300px` 可能已经影响了高度，比如文字换行变了，内容高度变了。

为了给你一个准确的 `offsetHeight`，浏览器只能立刻把之前挂起的样式和布局计算完成。

这就叫：

> 强制同步布局，或者强制回流。

---

**一个典型例子**

```
const box = document.querySelector('.box')

box.style.width = '300px'

const height = box.offsetHeight

box.style.height = height + 100 + 'px'
```

这里流程是：

```
1. 修改 width
2. 浏览器标记布局失效
3. 读取 offsetHeight
4. 浏览器被迫立即重新计算布局
5. 返回最新 height
6. 再修改 height
7. 后续又需要重新布局
```

如果这种逻辑在循环里，就很伤性能。

---

**哪些属性会触发强制布局**

常见会读取最新布局信息的属性包括：

```
el.offsetWidth
el.offsetHeight
el.offsetTop
el.offsetLeft

el.clientWidth
el.clientHeight
el.clientTop
el.clientLeft

el.scrollWidth
el.scrollHeight
el.scrollTop
el.scrollLeft

el.getBoundingClientRect()

window.getComputedStyle(el).width
```

这些属性要么关心尺寸，要么关心位置，要么关心滚动，要么关心最终计算样式。

浏览器为了返回准确值，可能会先完成布局计算。

注意措辞是：**可能会触发**。

如果当前布局本来就是干净的，没有待处理的样式修改，那么读取这些属性不一定产生额外回流。

真正危险的是这种模式：

```
写样式 -> 读布局 -> 写样式 -> 读布局
```

---

**最糟糕的写法：布局抖动**

比如：

```
const items = document.querySelectorAll('.item')

items.forEach(item => {
  item.style.width = '300px'
  console.log(item.offsetHeight)
})
```

每次循环都是：

```
写 style
读 offsetHeight
写 style
读 offsetHeight
...
```

浏览器可能被迫多次同步布局。

这叫：

```
layout thrashing
布局抖动
```

它会让页面卡顿。

---

**更好的写法：读写分离**

把所有读取布局的操作放一起，再统一写入样式。

差写法：

```
items.forEach(item => {
  item.style.width = '300px'
  const height = item.offsetHeight
  item.style.height = height + 20 + 'px'
})
```

好写法：

```
const heights = []

items.forEach(item => {
  heights.push(item.offsetHeight)
})

items.forEach((item, index) => {
  item.style.width = '300px'
  item.style.height = heights[index] + 20 + 'px'
})
```

更准确一点，如果改宽度会影响高度，那就应该先统一改宽度，再在下一帧读高度：

```
items.forEach(item => {
  item.style.width = '300px'
})

requestAnimationFrame(() => {
  const heights = Array.from(items, item => item.offsetHeight)

  items.forEach((item, index) => {
    item.style.height = heights[index] + 20 + 'px'
  })
})
```

这样浏览器有机会在帧边界统一处理布局。

---

**requestAnimationFrame 的作用**

`requestAnimationFrame` 会在浏览器下一次绘制前执行。

常用于把 DOM 操作安排到合适的渲染节奏里。

例如：

```
box.style.width = '300px'

requestAnimationFrame(() => {
  const rect = box.getBoundingClientRect()
  console.log(rect.height)
})
```

这样比立刻读：

```
box.style.width = '300px'
console.log(box.getBoundingClientRect())
```

更不容易造成频繁的同步布局。

但要注意：`requestAnimationFrame` 不是万能消除回流。它只是让你更容易把操作安排成：

```
一批写 -> 浏览器处理 -> 一批读/写
```

真正核心还是避免反复交替读写布局。

---

**getBoundingClientRect 为什么也会触发**

```
const rect = el.getBoundingClientRect()
```

它返回：

```
{
  x,
  y,
  width,
  height,
  top,
  right,
  bottom,
  left
}
```

这些都是元素相对于视口的位置和尺寸。

如果前面刚改了布局：

```
el.style.marginTop = '100px'
```

那 `top`、`bottom` 都变了。

所以浏览器必须先确保布局是最新的，才能返回准确的矩形信息。

---

**Vue / React 中也会遇到**

比如 Vue 里：

```
state.show = true

const height = panel.value.offsetHeight
```

这时候可能读不到你想要的最新 DOM，因为 Vue 更新 DOM 是异步的。

所以通常要：

```
import { nextTick } from 'vue'

state.show = true

await nextTick()

const height = panel.value.offsetHeight
```

但这里也有性能问题：

```
await nextTick()
const height = panel.value.offsetHeight
```

如果前面有大量 DOM 更新，读取高度依然可能迫使浏览器计算最新布局。

`nextTick` 解决的是：

```
等 Vue 把 DOM 更新完
```

不是解决：

```
完全避免回流
```

---

**如何减少回流**

常见优化：

1. **读写分离**

```
const rect = el.getBoundingClientRect()

el.style.width = rect.width + 20 + 'px'
```

避免在循环里交替读写。

2. **批量修改 class，而不是多次改 style**

```
el.classList.add('active')
```

比连续写很多 style 更容易维护，也方便浏览器优化。

3. **使用 transform 做动画**

差：

```
el.style.left = x + 'px'
```

好：

```
el.style.transform = `translateX(${x}px)`
```

`left/top/width/height` 容易触发布局，`transform/opacity` 通常只触发合成，性能更好。

4. **脱离文档流后再操作**

比如大量插入节点时，可以用 `DocumentFragment`：

```
const fragment = document.createDocumentFragment()

for (let i = 0; i < 1000; i++) {
  const li = document.createElement('li')
  li.textContent = i
  fragment.appendChild(li)
}

list.appendChild(fragment)
```

减少多次插入真实 DOM。

5. **复杂动画用 absolute/fixed 定位或独立图层**

减少对周围元素布局的影响。

重绘，也叫 `Repaint`，指的是：

> 元素的几何布局没有变化，但是外观样式变了，浏览器需要重新把它画出来。

也就是说，元素的：

```
位置没变
大小没变
占用空间没变
```

但视觉表现变了，比如颜色、背景、阴影等变了。

例如：

```
box.style.color = 'red'
box.style.backgroundColor = 'blue'
box.style.visibility = 'hidden'
```

这些通常会触发重绘，但不一定触发回流。

---

**回流和重绘的区别**

回流 `Reflow / Layout` 是重新计算布局。

比如：

```
box.style.width = '300px'
box.style.height = '200px'
box.style.marginTop = '20px'
```

这些会影响元素尺寸或位置，所以浏览器要重新算布局。

重绘 `Repaint` 是重新绘制外观。

比如：

```
box.style.backgroundColor = 'red'
box.style.color = '#333'
box.style.borderColor = 'blue'
```

这些不影响元素占据的空间，只影响它长什么样，所以通常只需要重新绘制。

可以这样记：

```
回流：重新算位置和大小
重绘：重新画颜色和外观
```

---

**一个例子**

HTML：

```
<div class="box">hello</div>
```

CSS：

```
.box {
  width: 100px;
  height: 100px;
  background: skyblue;
}
```

如果执行：

```
const box = document.querySelector('.box')

box.style.backgroundColor = 'red'
```

元素还是：

```
100px × 100px
```

位置也没变。

只是背景色从蓝色变成红色。

所以浏览器不需要重新计算布局，只需要重新绘制这个区域。

这就是重绘。

---

**哪些操作会触发重绘**

常见触发重绘的属性：

```
color
background
background-color
border-color
visibility
box-shadow
outline
text-decoration
```

比如：

```
el.style.color = 'red'
el.style.backgroundColor = '#000'
el.style.boxShadow = '0 0 10px rgba(0,0,0,.3)'
el.style.visibility = 'hidden'
```

这些通常不会改变布局，但会改变视觉效果。

---

**注意：有些属性可能既触发回流又触发重绘**

比如：

```
el.style.width = '200px'
```

宽度变了，浏览器先要重新计算布局。

布局变了之后，元素占据的位置也变了，最后还要重新绘制。

所以：

```
回流一定会伴随重绘
重绘不一定会伴随回流
```

这是很重要的一句话。

例如：

```
el.style.backgroundColor = 'red'
```

通常只是重绘。

但：

```
el.style.width = '200px'
```

通常是：

```
回流 + 重绘
```

---

**display:none 和 visibility:hidden 的区别**

这个经常和回流、重绘一起考。

```
el.style.display = 'none'
```

元素从文档流中消失，不再占据空间。

会影响布局，所以会触发回流。

```
el.style.visibility = 'hidden'
```

元素只是看不见，但仍然占据原来的空间。

不影响布局，通常只触发重绘。

所以：

```
display: none -> 回流 + 重绘
visibility: hidden -> 重绘
```

---

**opacity 和 transform 更特殊**

比如：

```
el.style.opacity = '0.5'
el.style.transform = 'translateX(100px)'
```

它们通常不会触发布局。

很多情况下，浏览器可以把元素提升到合成层，由 GPU 在合成阶段处理。

也就是说，它们可能连普通重绘都不需要，只需要合成。

大致可以理解为：

```
layout -> paint -> composite
```

`transform` 和 `opacity` 在很多动画场景里可以跳过 layout 和 paint，只走 composite。

所以动画性能优化里经常说：

> 尽量使用 `transform` 和 `opacity` 做动画，避免频繁改变 `left/top/width/height`。

比如差写法：

```
box.style.left = x + 'px'
```

可能触发回流。

好写法：

```
box.style.transform = `translateX(${x}px)`
```

通常只触发合成。

---

**重绘也可能很贵**

有些人会误以为：

> 重绘比回流轻，所以无所谓。

不完全对。

重绘虽然通常比回流轻，但如果绘制区域很大，或者样式很复杂，也会很耗性能。

比如：

```
box-shadow: 0 0 50px rgba(0, 0, 0, .5);
filter: blur(10px);
```

这类效果绘制成本高。

如果你在动画中频繁改变它们：

```
el.style.boxShadow = `0 0 ${size}px rgba(0,0,0,.5)`
```

可能会导致明显卡顿。

所以不仅要避免回流，也要避免大面积、复杂样式的频繁重绘。

---

**浏览器渲染流程里，重绘在哪一步**

简化流程：

```
JS 修改样式
  ↓
Style 样式计算
  ↓
Layout 回流，计算位置和尺寸
  ↓
Paint 重绘，把元素画成像素
  ↓
Composite 合成，把图层合成到屏幕
```

如果只改颜色：

```
el.style.color = 'red'
```

大概是：

```
Style -> Paint -> Composite
```

如果改宽度：

```
el.style.width = '200px'
```

大概是：

```
Style -> Layout -> Paint -> Composite
```

如果改 transform：

```
el.style.transform = 'translateX(100px)'
```

理想情况下是：

```
Style -> Composite
```

---

**面试版回答**

可以这样说：

> 重绘是指元素的布局没有变化，但是外观样式发生变化，浏览器需要重新绘制元素，比如修改 `color`、`background-color`、`visibility`、`box-shadow` 等。它和回流的区别是，回流会重新计算元素的位置和尺寸，而重绘只是重新绘制视觉效果。
> 
> 一般来说，回流一定会伴随重绘，因为布局变了以后页面需要重新画；但重绘不一定触发回流，比如只改背景色通常只会重绘。`display: none` 会让元素脱离文档流，所以会触发回流；`visibility: hidden` 只是隐藏但仍占位，所以通常只触发重绘。动画中尽量使用 `transform` 和 `opacity`，因为它们很多时候可以跳过布局和绘制，只在合成层处理，性能更好。

# 事件循环

你这里应该是想问：**浏览器事件循环机制 Event Loop**。这是前端面试高频题，而且经常和 `Promise`、`setTimeout`、`async/await` 一起考。

**一、为什么需要事件循环**

JavaScript 是**单线程**执行的。

也就是说，同一时间 JS 主线程只能做一件事：

```
console.log(1)
console.log(2)
console.log(3)
```

一定按顺序执行。

但浏览器里有很多异步任务：

```
setTimeout()
Promise.then()
DOM 事件
ajax 请求
script 脚本执行
页面渲染
```

如果 JS 只能一行一行同步执行，那异步任务完成后，什么时候回到 JS 主线程执行？

这就需要事件循环。

一句话：

> 事件循环是浏览器协调同步任务、异步任务、微任务、宏任务和页面渲染的一套调度机制。

---

**二、先记住几个核心概念**

浏览器事件循环里，最重要的是：

```
调用栈 Call Stack
宏任务 Macrotask / Task
微任务 Microtask
渲染 Render
事件循环 Event Loop
```

---

**调用栈 Call Stack**

同步代码会进入调用栈执行。

```
function foo() {
  console.log('foo')
}

function bar() {
  foo()
  console.log('bar')
}

bar()
```

执行顺序：

```
bar 入栈
foo 入栈
foo 执行完出栈
bar 执行完出栈
```

调用栈空了之后，事件循环才有机会处理异步任务。

---

**三、宏任务是什么**

常见宏任务：

```
script 整体代码
setTimeout
setInterval
setImmediate // Node 中
I/O
UI 事件，比如 click、keydown
postMessage
MessageChannel
```

浏览器一开始执行整个 JS 文件，本身就是一个宏任务：

```
console.log('script start')
```

可以理解为：

```
第一个宏任务：执行整段 script
```

---

**四、微任务是什么**

常见微任务：

```
Promise.then / catch / finally
queueMicrotask
MutationObserver
```

微任务的优先级比宏任务高。

也就是说：

> 当前宏任务执行完后，会先清空所有微任务，再去执行下一个宏任务。

---

**五、最核心执行顺序**

浏览器事件循环可以简化成：

```
1. 执行一个宏任务
2. 执行过程中遇到同步代码，直接执行
3. 遇到宏任务，放入宏任务队列
4. 遇到微任务，放入微任务队列
5. 当前宏任务执行完
6. 清空所有微任务
7. 必要时进行页面渲染
8. 执行下一个宏任务
9. 重复以上过程
```

关键句：

> 每执行完一个宏任务，都会清空当前所有微任务，然后浏览器才可能进行渲染，再进入下一个宏任务。

---

**六、经典题 1**

```
console.log(1)

setTimeout(() => {
  console.log(2)
}, 0)

Promise.resolve().then(() => {
  console.log(3)
})

console.log(4)
```

分析：

```
console.log(1)
```

同步执行，输出 `1`。

```
setTimeout(...)
```

放入宏任务队列。

```
Promise.then(...)
```

放入微任务队列。

```
console.log(4)
```

同步执行，输出 `4`。

当前 script 这个宏任务结束后，清空微任务，输出 `3`。

最后执行下一个宏任务，输出 `2`。

结果：

```
1
4
3
2
```

---

**七、经典题 2：微任务里继续产生微任务**

```
Promise.resolve().then(() => {
  console.log(1)

  Promise.resolve().then(() => {
    console.log(2)
  })
})

Promise.resolve().then(() => {
  console.log(3)
})

console.log(4)
```

先执行同步：

```
4
```

然后微任务队列最开始有两个：

```
then1
then3
```

执行 `then1`：

```
输出 1
产生新的微任务 then2，放到微任务队列末尾
```

此时微任务队列：

```
then3
then2
```

继续清空：

```
输出 3
输出 2
```

最终：

```
4
1
3
2
```

注意：

> 微任务队列必须清空，过程中新增的微任务也会在本轮继续执行。

---

**八、经典题 3：async / await**

`async/await` 本质上也是基于 Promise。

看这个：

```
async function fn() {
  console.log(1)
  await Promise.resolve()
  console.log(2)
}

console.log(3)

fn()

console.log(4)
```

执行：

```
console.log(3)
```

输出 `3`。

调用 `fn()`：

```
console.log(1)
```

输出 `1`。

遇到：

```
await Promise.resolve()
```

`await` 后面的代码会被放到微任务里，相当于：

```
Promise.resolve().then(() => {
  console.log(2)
})
```

然后继续执行外面的同步代码：

```
console.log(4)
```

输出 `4`。

同步结束后清空微任务，输出 `2`。

结果：

```
3
1
4
2
```

---

**九、setTimeout 为什么不是准确 0ms**

```
setTimeout(() => {
  console.log('timeout')
}, 0)
```

`0ms` 不是马上执行。

意思是：

> 最少等待 0ms 后，把回调放入宏任务队列。

但它要等当前宏任务执行完、微任务清空后，才有机会执行。

比如：

```
setTimeout(() => {
  console.log('timeout')
}, 0)

for (let i = 0; i < 1000000000; i++) {}

console.log('end')
```

输出一定是：

```
end
timeout
```

因为主线程被同步循环占着，`setTimeout` 回调只能等。

---

**十、事件循环和页面渲染的关系**

浏览器不是每执行一点 JS 就马上渲染页面。

大致是：

```
执行一个宏任务
清空微任务
浏览器判断是否需要渲染
执行渲染
执行下一个宏任务
```

比如：

```
const box = document.querySelector('.box')

box.style.width = '100px'

Promise.resolve().then(() => {
  box.style.width = '200px'
})

setTimeout(() => {
  box.style.width = '300px'
}, 0)
```

大概率页面不会先显示 `100px`，再显示 `200px`。

因为：

```
当前宏任务里改成 100px
微任务里改成 200px
微任务清空后才可能渲染
```

所以这一轮渲染时看到的可能已经是 `200px`。

而 `setTimeout` 是下一个宏任务，可能在之后一帧变成 `300px`。

---

**十一、为什么微任务太多会阻塞渲染**

因为浏览器要先清空微任务，再有机会渲染。

比如：

```
function loop() {
  Promise.resolve().then(loop)
}

loop()
```

这会不断产生微任务。

微任务队列一直清不完，浏览器就很难进入渲染阶段，页面可能卡死。

所以：

> 微任务优先级高，但不能滥用。

---

**十二、事件循环和用户点击**

比如用户点击按钮：

```
button.addEventListener('click', () => {
  console.log('click')
})
```

点击事件回调也是一个宏任务。

大概流程：

```
用户点击按钮
浏览器把 click 回调放入任务队列
主线程空闲后执行 click 回调
click 回调执行完
清空微任务
必要时渲染
继续下一个任务
```

---

**十三、完整例题**

```
console.log('start')

setTimeout(() => {
  console.log('timeout')
}, 0)

async function foo() {
  console.log('foo start')
  await Promise.resolve()
  console.log('foo end')
}

foo()

Promise.resolve().then(() => {
  console.log('promise')
})

console.log('end')
```

分析：

同步执行：

```
start
foo start
end
```

异步队列：

```
宏任务：timeout
微任务：foo await 后面的代码、promise.then
```

清空微任务：

```
foo end
promise
```

执行下一个宏任务：

```
timeout
```

最终：

```
start
foo start
end
foo end
promise
timeout
```

---

**十四、面试版回答**

可以这样说：

> 浏览器中的 JavaScript 是单线程执行的，事件循环用来协调同步代码、异步任务、微任务、宏任务以及页面渲染。同步代码会先进入调用栈执行，像整段 script、setTimeout、DOM 事件属于宏任务，Promise.then、queueMicrotask、MutationObserver 属于微任务。
> 
> 事件循环的基本流程是：先执行一个宏任务，执行过程中遇到同步代码直接执行，遇到宏任务放入宏任务队列，遇到微任务放入微任务队列；当前宏任务执行完后，会清空所有微任务，包括微任务执行过程中新增的微任务；然后浏览器可能进行页面渲染，接着再执行下一个宏任务。
> 
> 所以 Promise.then 通常会比 setTimeout 更早执行，因为它是微任务，而 setTimeout 回调是下一个宏任务。`async/await` 本质上也是 Promise，`await` 后面的代码可以理解为进入微任务队列。事件循环的重点就是：同步代码先执行，微任务在当前宏任务结束后立刻清空，宏任务一轮一轮执行。

# Node.js 事件循环
**Node.js 事件循环不是简单的宏任务队列**

Node.js 底层依赖 `libuv` 实现事件循环。

它不是只有一个宏任务队列，而是分成多个阶段。

常见阶段可以简化为：

```
timers
  ↓
pending callbacks
  ↓
idle / prepare
  ↓
poll
  ↓
check
  ↓
close callbacks
```

你重点记这几个就够了：

|阶段|主要处理|
|---|---|
|`timers`|`setTimeout`、`setInterval`|
|`poll`|I/O 回调，比如文件读取、网络请求|
|`check`|`setImmediate`|
|`close callbacks`|socket close 等关闭回调|

所以 Node.js 的事件循环更像：

```
进入某个阶段
执行该阶段队列里的回调
清空 nextTick 队列
清空 Promise 微任务队列
进入下一个阶段
```

---

**三、Node.js 里的微任务有两类**

Node 里要特别注意：

```
process.nextTick()
Promise.then()
queueMicrotask()
```

其中：

```
process.nextTick()
```

不是标准浏览器 API，它是 Node 自己的。

它的优先级通常比 Promise 微任务还高。

看例子：

```
console.log('start')

Promise.resolve().then(() => {
  console.log('promise')
})

process.nextTick(() => {
  console.log('nextTick')
})

console.log('end')
```

Node.js 输出通常是：

```
start
end
nextTick
promise
```

因为当前同步代码执行完后，Node 会先清空 `nextTick` 队列，再执行 Promise 微任务队列。

可以记成：

```
同步代码
  ↓
process.nextTick
  ↓
Promise / queueMicrotask
  ↓
下一轮事件循环阶段
```

---

**四、setTimeout 和 setImmediate 的区别**

Node 面试很爱问：

```
setTimeout(() => {
  console.log('timeout')
}, 0)

setImmediate(() => {
  console.log('immediate')
})
```

这个输出顺序在主模块里不绝对稳定，可能是：

```
timeout
immediate
```

也可能是：

```
immediate
timeout
```

原因是：

- `setTimeout(fn, 0)` 在 `timers` 阶段执行
- `setImmediate(fn)` 在 `check` 阶段执行
- 如果代码在主模块中运行，受启动耗时和计时器是否到期影响，顺序可能不固定

但是如果放在 I/O 回调里，顺序通常比较明确：

```
const fs = require('fs')

fs.readFile('./test.txt', () => {
  setTimeout(() => {
    console.log('timeout')
  }, 0)

  setImmediate(() => {
    console.log('immediate')
  })
})
```

通常输出：

```
immediate
timeout
```

因为 I/O 回调在 `poll` 阶段执行，执行完后会进入 `check` 阶段，所以 `setImmediate` 先执行。`setTimeout` 要等下一轮的 `timers` 阶段。

---

**五、浏览器有渲染，Node 没有页面渲染**

这是很大的区别。

浏览器事件循环里要考虑：

```
宏任务 -> 微任务 -> 渲染 -> 下一个宏任务
```

比如 DOM 更新后，浏览器可能在一轮事件循环末尾进行渲染。

但 Node.js 没有 DOM，也没有页面渲染阶段。它关心的是：

```
定时器
I/O
网络
文件
进程
流
```

所以 Node 的事件循环重点是各个 libuv 阶段。

---

**六、Node 的 I/O 更核心**

浏览器里的异步很多和用户交互有关：

```
click
scroll
setTimeout
Promise
ajax
requestAnimationFrame
```

Node.js 里的异步大量来自 I/O：

```
fs.readFile
http request
net socket
database query
stream
```

比如：

```
const fs = require('fs')

fs.readFile('./a.txt', () => {
  console.log('file read done')
})
```

文件读取交给底层线程池或系统能力处理，完成后回调会进入事件循环的对应阶段，等 JS 主线程空闲后执行。

---

**七、一个综合例子**

```
console.log('start')

setTimeout(() => {
  console.log('timeout')
}, 0)

setImmediate(() => {
  console.log('immediate')
})

Promise.resolve().then(() => {
  console.log('promise')
})

process.nextTick(() => {
  console.log('nextTick')
})

console.log('end')
```

Node.js 中常见输出：

```
start
end
nextTick
promise
timeout / immediate
immediate / timeout
```

前四个比较确定：

```
start
end
nextTick
promise
```

后面 `timeout` 和 `immediate` 在主模块中顺序不完全稳定。

---

**八、核心区别总结**

|对比点|浏览器|Node.js|
|---|---|---|
|主要目标|页面交互、DOM、渲染|I/O、网络、文件、服务端任务|
|事件循环模型|宏任务 + 微任务 + 渲染|libuv 多阶段事件循环|
|微任务|`Promise`、`queueMicrotask`、`MutationObserver`|`process.nextTick`、`Promise`、`queueMicrotask`|
|特殊队列|无 `process.nextTick`|`process.nextTick` 优先级很高|
|渲染阶段|有|没有|
|定时器|`setTimeout`、`setInterval`|`timers` 阶段处理|
|`setImmediate`|浏览器一般没有|`check` 阶段处理|
|I/O|ajax、事件等|文件、网络、stream，更核心|

---

**面试版回答**

可以这样说：

> 浏览器和 Node.js 都有事件循环，但侧重点不同。浏览器事件循环主要围绕宏任务、微任务和页面渲染展开，执行完一个宏任务后会清空微任务，然后浏览器可能进行渲染，再执行下一个宏任务。
> 
> Node.js 的事件循环基于 libuv，被分成多个阶段，比如 `timers`、`poll`、`check`、`close callbacks`。`setTimeout` 在 timers 阶段执行，I/O 回调主要在 poll 阶段执行，`setImmediate` 在 check 阶段执行。Node 还多了 `process.nextTick` 队列，它的优先级通常高于 Promise 微任务。
> 
> 所以浏览器重点关注渲染和用户交互，Node 重点关注 I/O 调度；浏览器没有 `process.nextTick` 和标准的 `setImmediate`，Node 也没有浏览器的页面渲染阶段。