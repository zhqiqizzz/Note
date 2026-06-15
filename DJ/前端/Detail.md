
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


# 用户打开 URL 到浏览器完全渲染的全过程

这是前端面试的超级高频题。建议你不要死背“DNS、TCP、HTTP、渲染”几个词，而是按一条主线讲：

> 输入 URL 后，浏览器先找到服务器，拿到资源，然后解析资源，构建页面，最后渲染到屏幕上。

可以分成 **网络阶段** 和 **渲染阶段**。

**一、用户输入 URL**

比如用户输入：

```
https://www.example.com/index.html
```

浏览器会先解析 URL：

```
协议：https
域名：www.example.com
路径：/index.html
端口：默认 443
```

如果用户只输入：

```
www.example.com
```

浏览器还会补全协议，或者根据历史记录、搜索引擎策略判断是搜索还是访问网址。

---

**二、检查缓存**

真正发请求前，浏览器会先看有没有缓存。

常见缓存包括：

```
强缓存
协商缓存
DNS 缓存
Service Worker 缓存
```

如果命中强缓存，比如：

```
Cache-Control: max-age=3600
```

浏览器可能直接从本地缓存读取资源，不发请求。

如果强缓存过期，浏览器会发请求给服务器做协商缓存，比如带上：

```
If-None-Match
If-Modified-Since
```

服务器如果返回：

```
304 Not Modified
```

浏览器继续用本地缓存。

如果缓存没命中，才会进入真正网络请求。

---

**三、DNS 解析**

浏览器要访问的是域名：

```
www.example.com
```

但网络通信需要 IP 地址。

所以要做 DNS 解析，把域名解析成 IP：

```
www.example.com -> 93.184.216.34
```

DNS 查询大概会经过：

```
浏览器 DNS 缓存
系统 DNS 缓存
本地 hosts 文件
路由器 / 本地 DNS 服务器
根域名服务器
顶级域名服务器
权威 DNS 服务器
```

实际中如果缓存命中，就不会走完整链路。

---

**四、建立 TCP 连接**

拿到 IP 后，如果是 HTTP/1.1 或 HTTP/2，底层通常要建立 TCP 连接。

TCP 连接需要三次握手：

```
客户端 -> SYN -> 服务端
服务端 -> SYN + ACK -> 客户端
客户端 -> ACK -> 服务端
```

三次握手完成后，客户端和服务端之间就有了可靠连接。

---

**五、TLS 握手**

如果是 HTTPS，还需要进行 TLS 握手。

HTTPS = HTTP + TLS。

TLS 握手大概做几件事：

```
协商加密算法
验证服务器证书
生成会话密钥
建立安全加密通道
```

这样后续 HTTP 请求和响应就会被加密传输。

所以 HTTPS 比 HTTP 多了 TLS 握手阶段，但现代浏览器会通过连接复用、TLS 1.3、会话恢复等方式优化。

---

**六、发送 HTTP 请求**

连接建立后，浏览器发送 HTTP 请求。

比如：

```
GET /index.html HTTP/1.1
Host: www.example.com
Cookie: xxx
User-Agent: xxx
Accept: text/html
```

如果是登录态页面，浏览器还会根据同源、路径、`SameSite` 等规则携带 cookie。

---

**七、服务器处理请求并返回响应**

服务器收到请求后，可能经过：

```
Nginx
后端应用
数据库
缓存 Redis
文件系统
CDN
```

然后返回 HTTP 响应：

```
HTTP/1.1 200 OK
Content-Type: text/html
Cache-Control: max-age=3600
Content-Encoding: gzip

<html>...</html>
```

如果资源不存在，可能返回：

```
404 Not Found
```

如果重定向，可能返回：

```
301 / 302
Location: https://example.com/new
```

浏览器会根据状态码继续处理。

---

**八、浏览器接收 HTML，开始解析**

浏览器拿到 HTML 后，不是等全部下载完才开始处理，而是边下载边解析。

HTML 解析成：

```
DOM Tree
```

比如：

```
<div>
  <p>Hello</p>
</div>
```

会变成 DOM 节点树：

```
div
 └─ p
    └─ text
```

---

**九、解析 CSS，构建 CSSOM**

遇到 CSS：

```
<link rel="stylesheet" href="/style.css">
```

浏览器会下载 CSS，并解析成：

```
CSSOM Tree
```

CSSOM 记录每个选择器和样式规则。

注意：

> CSS 会阻塞渲染。

因为浏览器必须知道元素最终样式，才能准确绘制页面。

---

**十、遇到 JS 怎么办**

HTML 解析过程中如果遇到普通 script：

```
<script src="/main.js"></script>
```

默认会阻塞 HTML 解析。

流程是：

```
暂停 HTML 解析
下载 JS
执行 JS
继续解析 HTML
```

为什么？

因为 JS 可能会修改 DOM：

```
document.write()
document.querySelector('div').remove()
```

所以浏览器必须停下来执行它。

如果是：

```
<script defer src="/main.js"></script>
```

JS 下载不阻塞 HTML 解析，等 DOM 解析完成后按顺序执行。

如果是：

```
<script async src="/main.js"></script>
```

JS 下载不阻塞 HTML 解析，但下载完成后立刻执行，可能打断 HTML 解析。

如果是：

```
<script type="module" src="/main.js"></script>
```

默认类似 `defer`，并且支持 ESM。

---

**十一、构建渲染树 Render Tree**

浏览器有了 DOM 和 CSSOM 后，会合成：

```
Render Tree
```

Render Tree 只包含需要显示的节点。

比如：

```
display: none;
```

对应元素不会进入渲染树。

但：

```
visibility: hidden;
```

元素虽然看不见，但还占位置，一般仍然会参与布局。

---

**十二、布局 Layout / Reflow重排**

有了 Render Tree 后，浏览器开始计算布局：

```
每个元素的位置
每个元素的宽高
文字如何换行
盒模型大小
元素之间怎么排列
```

比如：

```
.box {
  width: 50%;
  padding: 20px;
}
```

浏览器要根据视口宽度、父元素大小、盒模型规则算出实际像素尺寸。

这一步叫：

```
Layout / Reflow
```

---

**十三、绘制 Paint 重绘**

布局完成后，浏览器知道每个元素的位置和大小。

接下来要绘制：

```
文字
颜色
背景
边框
阴影
图片
```

这一步叫：

```
Paint / Repaint
```

浏览器会生成绘制指令，告诉后续阶段每一层该画什么。

---

**十四、合成 Composite**

现代浏览器会把页面分成多个图层。

比如这些元素可能被提升为单独图层：

```
transform
opacity
position: fixed
will-change
video
canvas
```

合成阶段会把多个图层组合成最终画面，显示到屏幕上。

这一步叫：

```
Composite
```

这也是为什么动画推荐用：

```
transform
opacity
```

因为它们很多时候可以跳过布局和绘制，只在合成阶段处理，性能更好。

---

**十五、加载其他资源**

在解析 HTML 的过程中，浏览器还会发现并下载其他资源：

```
<img src="/logo.png">
<script src="/main.js">
<link rel="stylesheet" href="/style.css">
<link rel="preload" href="/font.woff2">
```

浏览器会根据资源优先级调度请求。

比如：

- CSS 优先级高，因为阻塞渲染
- 首屏图片优先级较高
- 懒加载图片优先级低
- async 脚本不会阻塞 DOM 解析

---

**十六、DOMContentLoaded 和 load**

两个事件也经常问。

`DOMContentLoaded`：

> HTML 被完整解析，DOM 树构建完成，并且 defer/module 脚本执行完成后触发。不一定等图片、视频等资源加载完成。

`load`：

> 页面所有资源都加载完成后触发，包括图片、CSS、JS、字体等。

通常：

```
DOMContentLoaded 早于 load
```

---

**十七、完整流程总结**

可以这样串起来：

```
输入 URL
  ↓
浏览器解析 URL
  ↓
检查缓存
  ↓
DNS 解析域名得到 IP
  ↓
建立 TCP 连接
  ↓
HTTPS 进行 TLS 握手
  ↓
发送 HTTP 请求
  ↓
服务器处理请求并返回响应
  ↓
浏览器接收 HTML
  ↓
解析 HTML 构建 DOM
  ↓
解析 CSS 构建 CSSOM
  ↓
执行 JS，可能阻塞 DOM 解析
  ↓
DOM + CSSOM 生成 Render Tree
  ↓
Layout 计算位置和大小
  ↓
Paint 绘制像素
  ↓
Composite 合成图层
  ↓
页面展示完成
```

---

**面试版回答**

你可以这样说：

> 用户输入 URL 后，浏览器首先会解析 URL，并检查缓存。如果缓存没有命中，就进行 DNS 解析，把域名解析成 IP；然后建立 TCP 连接，如果是 HTTPS，还会进行 TLS 握手。连接建立后浏览器发送 HTTP 请求，服务器处理后返回 HTML。
> 
> 浏览器拿到 HTML 后会边下载边解析，构建 DOM 树；遇到 CSS 会下载并解析成 CSSOM，CSS 会阻塞渲染；遇到普通 script 会暂停 HTML 解析，下载并执行 JS，因为 JS 可能修改 DOM。DOM 和 CSSOM 构建完成后，会生成 Render Tree，然后进入 Layout 阶段计算元素的位置和大小，再进入 Paint 阶段绘制颜色、文字、图片、边框等，最后通过 Composite 合成图层并显示到屏幕上。
> 
> 在这个过程中，`DOMContentLoaded` 表示 DOM 解析完成并且 defer/module 脚本执行完成，`load` 表示页面所有资源，包括图片、样式、脚本等都加载完成。

# 跨域请求想传递 cookie

跨域请求想传递 cookie，要同时满足 **前端、后端、cookie 属性** 三边配置。少一个都不行。

**一、前端要开启携带凭证**

如果用 `fetch`：

```
fetch('https://api.example.com/user', {
  method: 'GET',
  credentials: 'include'
})
```

关键是：

```
credentials: 'include'
```

默认情况下，`fetch` 跨域请求不会带 cookie。

如果用 `axios`：

```
axios.get('https://api.example.com/user', {
  withCredentials: true
})
```

或者全局配置：

```
axios.defaults.withCredentials = true
```

关键是：

```
withCredentials: true
```

---

**二、后端 CORS 要允许携带凭证**

后端响应头必须有：

```
Access-Control-Allow-Credentials: true
```

同时还要指定明确的 Origin：

```
Access-Control-Allow-Origin: https://www.example.com
```

不能写：

```
Access-Control-Allow-Origin: *
```

因为携带 cookie 时，浏览器不允许 `Access-Control-Allow-Origin` 是 `*`。

正确示例：

```
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Credentials: true
```

错误示例：

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

---

**三、cookie 本身要允许跨站发送**

现在浏览器对第三方 cookie 管得很严。

如果是跨站请求，服务端设置 cookie 时通常需要：

```
Set-Cookie: token=abc123; Path=/; SameSite=None; Secure; HttpOnly
```

关键是：

```
SameSite=None
Secure
```

含义：

- `SameSite=None`：允许跨站请求携带 cookie
- `Secure`：只能在 HTTPS 下发送
- `HttpOnly`：JS 不能通过 `document.cookie` 读取，减少 XSS 风险

注意：

> 如果设置了 `SameSite=None`，现代浏览器要求必须同时设置 `Secure`，也就是必须 HTTPS。

---

**四、完整流程示例**

假设前端是：

```
https://www.example.com
```

后端是：

```
https://api.example.com
```

前端请求：

```
fetch('https://api.example.com/profile', {
  credentials: 'include'
})
```

后端响应：

```
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Credentials: true
Content-Type: application/json
```

服务端设置 cookie：

```
Set-Cookie: sid=xxx; Path=/; SameSite=None; Secure; HttpOnly
```

之后浏览器再次请求：

```
GET /profile HTTP/1.1
Host: api.example.com
Cookie: sid=xxx
Origin: https://www.example.com
```

---

**五、如果有预检请求**

如果是复杂请求，比如：

```
fetch('https://api.example.com/user', {
  method: 'POST',
  credentials: 'include',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'Tom' })
})
```

浏览器可能先发 `OPTIONS` 预检请求。

后端也要正确响应：

```
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET,POST,PUT,DELETE,OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
```

否则真正的请求不会发出去。

---

**六、常见坑**

1. **前端忘了加 `withCredentials` / `credentials`**

```
axios.get(url)
```

这样跨域默认不带 cookie。

2. **后端写了 `Access-Control-Allow-Origin: *`**

携带 cookie 时不能用 `*`。

3. **cookie 没有设置 `SameSite=None; Secure`**

跨站请求时 cookie 可能不会被发送。

4. **本地 HTTP 环境测试失败**

因为 `SameSite=None` 要求 `Secure`，而 `Secure` 要求 HTTPS。

本地开发经常因此踩坑。

5. **cookie 的 Domain / Path 不匹配**

比如 cookie 设置在：

```
Domain=api.example.com
```

那它不会发给：

```
www.example.com
```

cookie 是根据目标请求域名决定是否携带的，不是根据前端页面域名。

6. **把跨域和跨站混淆**

跨域是协议、域名、端口任一不同。

跨站主要看站点，比如 eTLD+1。

例如：

```
https://www.example.com
https://api.example.com
```

它们跨域，但通常是同站，因为主站点都是 `example.com`。

SameSite 判断的是“站”，不是严格的 origin。

---

**七、面试版回答**

可以这样说：

> 跨域请求携带 cookie 需要前后端同时配置。前端如果用 axios，需要设置 `withCredentials: true`；如果用 fetch，需要设置 `credentials: 'include'`。后端需要返回 `Access-Control-Allow-Credentials: true`，并且 `Access-Control-Allow-Origin` 不能是 `*`，必须是具体的前端域名。
> 
> 同时 cookie 本身也要满足浏览器策略。如果是跨站请求，服务端设置 cookie 时需要加上 `SameSite=None; Secure`，并且通常要在 HTTPS 下才会生效。还要注意 cookie 的 Domain、Path 是否和请求目标匹配。如果是复杂请求，还需要正确处理 OPTIONS 预检请求。


# `SameSite=Lax`

> `SameSite=Lax` 在很多“同站跨域”场景下可以携带 cookie；但在真正“跨站”的 AJAX / fetch / axios 请求里，一般不行。真正第三方跨站请求通常需要 `SameSite=None; Secure`。

---

**一、先区分跨域和跨站**

跨域看的是 origin：

```
协议 + 域名 + 端口
```

只要有一个不同，就是跨域。

例如：

```
https://www.example.com
https://api.example.com
```

这是跨域，因为域名不同。

但它们通常是同站，因为主站点都是：

```
example.com
```

跨站看的是 site，简单理解是：

```
协议 + 可注册域名
```

例如：

```
https://www.example.com
https://api.example.com
```

一般是同站。

但：

```
https://www.example.com
https://api.other.com
```

这是跨站。

`SameSite` 管的是**跨站 cookie 发送策略**，不是严格意义上的跨域策略。

---

**二、SameSite=Lax 的规则**

`SameSite=Lax` 大概意思是：

> 大多数跨站子请求不带 cookie，但跨站的顶级导航 GET 请求可以带 cookie。

什么叫顶级导航？

比如用户点击链接：

```
<a href="https://api.example.com/page">跳转</a>
```

或者在地址栏直接访问：

```
https://api.example.com/page
```

这种是顶级导航。

如果 cookie 是 `SameSite=Lax`，这种跨站 GET 导航请求通常可以带 cookie。

---

但下面这种不是顶级导航，而是子资源 / AJAX 请求：

```
fetch('https://api.other.com/user', {
  credentials: 'include'
})
```

或者：

```
<img src="https://api.other.com/a.png">
```

或者：

```
<form method="POST" action="https://api.other.com/login">
```

这类跨站请求，`SameSite=Lax` 通常不会携带 cookie。

---

**三、举例说明**

假设前端页面是：

```
https://www.a.com
```

接口是：

```
https://api.b.com
```

服务端设置：

```
Set-Cookie: sid=abc; SameSite=Lax; Secure; HttpOnly
```

然后前端发：

```
fetch('https://api.b.com/user', {
  credentials: 'include'
})
```

这通常不会带 `sid`。

因为这是：

```
跨站 + fetch 子请求
```

`SameSite=Lax` 会限制它。

如果想让这种请求带 cookie，通常需要：

```
Set-Cookie: sid=abc; SameSite=None; Secure; HttpOnly
```

---

**四、那什么情况下 Lax 可以？**

比如页面在：

```
https://www.example.com
```

接口在：

```
https://api.example.com
```

这是：

```
跨域，但同站
```

如果 cookie 的 Domain 设置合理，比如：

```
Set-Cookie: sid=abc; Domain=.example.com; Path=/; SameSite=Lax; Secure; HttpOnly
```

那么请求：

```
fetch('https://api.example.com/user', {
  credentials: 'include'
})
```

在 SameSite 这一层通常是允许的。

但仍然要满足 CORS：

```
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Credentials: true
```

前端也要设置：

```
credentials: 'include'
```

或 axios：

```
withCredentials: true
```

所以这里要分清：

```
SameSite=Lax 不是用来解决 CORS 的
它只决定跨站时 cookie 能不能发
```

---

**五、Lax / None / Strict 对比**

|SameSite|行为|
|---|---|
|`Strict`|只有同站请求才带 cookie，跨站基本不带|
|`Lax`|同站请求带；跨站顶级 GET 导航可能带；跨站 AJAX 通常不带|
|`None`|允许跨站请求携带 cookie，但必须配合 `Secure`|

现代浏览器如果不写 SameSite，很多情况下默认按：

```
SameSite=Lax
```

# 什么是简单请求？什么是复杂请求？

这是 CORS 里的概念。浏览器会先判断一次跨域请求是不是**简单请求**：

- 如果是简单请求：直接发真正请求。
- 如果是复杂请求：先发一次 `OPTIONS` 预检请求，确认服务器允许后，再发真正请求。

**简单请求**

同时满足下面几个条件，才是简单请求。

**1. 请求方法只能是这三个之一**

```
GET
POST
HEAD
```

比如：

```
fetch('https://api.example.com/user')
```

`GET` 可以。

```
fetch('https://api.example.com/user', {
  method: 'DELETE'
})
```

`DELETE` 不行，会变成复杂请求。

---

**2. 请求头只能包含 CORS 安全名单里的字段**

常见允许的请求头包括：

```
Accept
Accept-Language
Content-Language
Content-Type
Range
```

但这里有细节，`Content-Type` 也不是随便什么值都行。

比如你自己加了：

```
headers: {
  Authorization: 'Bearer xxx'
}
```

这就不是简单请求了，会触发预检。

再比如：

```
headers: {
  'X-Token': 'abc'
}
```

自定义请求头也会触发预检。

---

**3. Content-Type 只能是这三种之一**

如果有 `Content-Type`，只能是：

```
application/x-www-form-urlencoded
multipart/form-data
text/plain
```

例如：

```
fetch('https://api.example.com/login', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: 'name=Tom&age=18'
})
```

这是简单请求。

但这个不是：

```
fetch('https://api.example.com/login', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'Tom' })
})
```

因为：

```
Content-Type: application/json
```

不在简单请求允许范围内，所以会触发预检，属于复杂请求。

---

**复杂请求**

只要不满足简单请求条件，就是复杂请求。

常见复杂请求：

```
fetch('https://api.example.com/user', {
  method: 'PUT'
})
```

因为方法是 `PUT`。

```
fetch('https://api.example.com/user', {
  method: 'DELETE'
})
```

因为方法是 `DELETE`。

```
fetch('https://api.example.com/user', {
  headers: {
    Authorization: 'Bearer xxx'
  }
})
```

因为带了 `Authorization` 请求头。

```
fetch('https://api.example.com/user', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'Tom' })
})
```

因为 `Content-Type` 是 `application/json`。

---

**复杂请求会发生什么**

浏览器会先发一个 `OPTIONS` 预检请求。

比如你写：

```
fetch('https://api.example.com/user', {
  method: 'PUT',
  headers: {
    Authorization: 'Bearer xxx',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'Tom' })
})
```

浏览器不会马上发 `PUT`。

它会先发：

```
OPTIONS /user HTTP/1.1
Origin: https://www.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: authorization, content-type
```

意思是问服务器：

```
我这个来源 https://www.example.com
想用 PUT 方法
还想带 authorization 和 content-type 请求头
你允许吗？
```

服务器如果允许，要返回类似：

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Allow-Credentials: true
```

浏览器看到允许后，才会继续发送真正的 `PUT` 请求。

如果服务器预检没通过，真正请求不会发出去。

---

**简单请求也需要 CORS 响应头吗**

需要。

简单请求只是“不需要预检”，不是“不需要 CORS”。

比如：

```
fetch('https://api.example.com/user')
```

浏览器会直接发真正请求。

但响应里仍然需要：

```
Access-Control-Allow-Origin: https://www.example.com
```

否则浏览器会拦截响应，前端 JS 读不到结果。

所以：

```
简单请求：直接发请求，但响应仍需 CORS 允许
复杂请求：先发 OPTIONS 预检，再发真正请求
```

---

**如果要带 cookie**

无论简单还是复杂，只要跨域带 cookie，都要：

前端：

```
fetch(url, {
  credentials: 'include'
})
```

后端：

```
Access-Control-Allow-Origin: https://www.example.com
Access-Control-Allow-Credentials: true
```

并且不能：

```
Access-Control-Allow-Origin: *
```

如果是复杂请求，预检响应里也要正确处理这些头。

---

**面试版回答**

可以这样说：

> 简单请求和复杂请求是浏览器 CORS 里的分类。简单请求要求方法只能是 `GET`、`POST`、`HEAD`，请求头只能是 CORS 安全名单里的头，并且 `Content-Type` 只能是 `application/x-www-form-urlencoded`、`multipart/form-data` 或 `text/plain`。满足这些条件的跨域请求会直接发送真正请求。
> 
> 只要不满足这些条件，就是复杂请求，比如使用 `PUT`、`DELETE`，或者带 `Authorization`、自定义请求头，或者 `Content-Type` 是 `application/json`。复杂请求在真正请求前，浏览器会先发送 `OPTIONS` 预检请求，询问服务器是否允许该来源、方法和请求头。预检通过后，浏览器才会发送真正请求。
> 
> 需要注意，简单请求只是不会触发预检，不代表不需要 CORS。无论简单还是复杂，服务器响应都必须带正确的 `Access-Control-Allow-Origin`，否则浏览器仍然会拦截响应。

# Web性能优化

Web 优化是个很大的话题，面试里不要只背技术名词，要能说清**为什么这样做、什么场景用、怎么量化效果**。

可以从几个维度讲：**加载、渲染、代码、构建、网络、监控**。

---

**一、加载优化**

**1. 减少资源体积**

代码压缩：

```
JS / CSS / HTML 压缩
去除注释、空格、多余换行
```

工具链一般会自动做，比如 Vite / Webpack 生产构建。

图片优化：

```
使用 WebP / AVIF
压缩图片
雪碧图合并小图标(用SVG Sprite `<symbol>` + `<use>`)
懒加载非首屏图片
```

比如：

```
<img src="placeholder.jpg" data-src="real.jpg" loading="lazy">
```

浏览器原生 `loading="lazy"` 可以做懒加载。

字体优化：

```
@font-face {
  font-display: swap;
}
```

避免字体加载阻塞渲染。

---

**2. 减少请求数量**

合并资源：

```
CSS / JS 打包
雪碧图
内联关键 CSS
```

但注意不要过度合并，否则缓存失效时要重新下载整个大文件。

HTTP/2 多路复用后，不一定要极端合并小文件。

使用 CDN：

```
<script src="https://cdn.jsdelivr.net/npm/vue@3"></script>
```

CDN 可以：

```
离用户更近，延迟更低
分担服务器压力
通常有更好的缓存策略
```

---

**3. 利用缓存**

强缓存：

```
Cache-Control: max-age=31536000
```

适合不变的静态资源，比如打了 hash 的 JS / CSS。

协商缓存：

```
ETag
Last-Modified
```

资源变化时才重新下载。

Service Worker 缓存：

```
self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request).then(response => {
      return response || fetch(event.request)
    })
  )
})
```

可以做离线访问、秒开页面。

---

**4. 预加载 / 预连接**

```
<link rel="preload" href="/critical.css" as="style">
<link rel="prefetch" href="/next-page.js">
<link rel="preconnect" href="https://api.example.com">
<link rel="dns-prefetch" href="https://cdn.example.com">
```

`preload`：提前加载关键资源。

`prefetch`：空闲时加载下一页资源。

`preconnect`：提前建立连接。

`dns-prefetch`：提前做 DNS 解析。

---

**二、渲染优化**

**1. 减少回流和重绘**

避免频繁读写布局：

差：

```
items.forEach(item => {
  item.style.width = '300px'
  const height = item.offsetHeight
  item.style.height = height + 20 + 'px'
})
```

好：

```
const heights = items.map(item => item.offsetHeight)

items.forEach((item, i) => {
  item.style.width = '300px'
  item.style.height = heights[i] + 20 + 'px'
})
```

读写分离，避免 layout thrashing。

批量修改 DOM：

```
const fragment = document.createDocumentFragment()

for (let i = 0; i < 1000; i++) {
  const li = document.createElement('li')
  li.textContent = i
  fragment.appendChild(li)
}

list.appendChild(fragment)
```

避免多次触发布局。

---

**2. 使用 transform / opacity 做动画**

差：

```
@keyframes move {
  to {
    left: 100px;
  }
}
```

好：

```
@keyframes move {
  to {
    transform: translateX(100px);
  }
}
```

`transform` 和 `opacity` 通常只触发合成，不触发布局和绘制，性能更好。

---

**3. 虚拟列表**

如果要渲染几千条数据：

```
只渲染可视区域的内容
滚动时动态更新渲染内容
```

常用库：

```
react-window
vue-virtual-scroller
```

比如电商商品列表、评论区、长表格。

---

**4. 避免长任务阻塞主线程**

如果有大量计算：

```
for (let i = 0; i < 100000000; i++) {
  // 复杂计算
}
```

会阻塞页面。

可以拆成小任务：

```
function processChunk(i, max) {
  const end = Math.min(i + 1000, max)
  for (let j = i; j < end; j++) {
    // 处理
  }
  if (end < max) {
    setTimeout(() => processChunk(end, max), 0)
  }
}

processChunk(0, 100000000)
```

或者用 Web Worker：

```
const worker = new Worker('/worker.js')
worker.postMessage(data)
worker.onmessage = e => {
  console.log(e.data)
}
```

Worker 在独立线程执行，不阻塞主线程。

---

**三、代码层面优化**

**1. Tree Shaking**

只打包用到的代码。

比如你写：

```
import { debounce } from 'lodash-es'
```

Webpack / Vite 会只打包 `debounce`，不会把整个 lodash 都打包进去。

前提是模块是 ESM，并且生产环境开启压缩。

---

**2. 代码分割**

把代码拆成多个小文件，按需加载。

Vue Router：

```
const Home = () => import('./views/Home.vue')
const About = () => import('./views/About.vue')

const routes = [
  { path: '/', component: Home },
  { path: '/about', component: About }
]
```

React Router：

```
const Home = lazy(() => import('./Home'))
```

这样首屏只加载当前路由代码，减少初始包体积。

---

**3. 防抖和节流**

频繁触发的事件，比如 `scroll`、`resize`、`input`，做防抖或节流。

防抖：

```
const debounce = (fn, delay) => {
  let timer
  return (...args) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}

input.addEventListener('input', debounce(() => {
  console.log('search')
}, 300))
```

节流：

```
const throttle = (fn, delay) => {
  let last = 0
  return (...args) => {
    const now = Date.now()
    if (now - last >= delay) {
      fn(...args)
      last = now
    }
  }
}

window.addEventListener('scroll', throttle(() => {
  console.log('scroll')
}, 100))
```

---

**4. 避免内存泄漏**

常见泄漏场景：

```
// 忘记取消定时器
const timer = setInterval(() => {}, 1000)

// 忘记移除事件监听
window.addEventListener('scroll', handler)

// 闭包引用
function leak() {
  const bigData = new Array(1000000)
  return () => {
    console.log(bigData.length)
  }
}
```

Vue / React 组件卸载时要清理：

```
onUnmounted(() => {
  clearInterval(timer)
  window.removeEventListener('scroll', handler)
})
```

---

**四、构建优化**

**1. 生产环境压缩**

Vite / Webpack 默认会做：

```
JS 压缩：terser / esbuild
CSS 压缩：cssnano
HTML 压缩
```

确保生产构建时开启：

```
mode: 'production'
```

---

**2. 分包策略**

常见分包：

```
vendor：第三方库
common：多个页面共用代码
page：页面独有代码
```

Webpack 示例：

```
optimization: {
  splitChunks: {
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendor',
        chunks: 'all'
      }
    }
  }
}
```

这样第三方库变化少，可以长期缓存。

---

**3. 开启 gzip / brotli**

服务端开启压缩：

```
gzip on;
gzip_types text/plain text/css application/json application/javascript;
```

响应头：

```
Content-Encoding: gzip
```

通常能减少 60%~80% 传输体积。

---

**五、网络优化**

**1. HTTP/2**

支持多路复用、头部压缩、服务端推送。

配置 HTTPS + HTTP/2 可以明显提升并发资源加载速度。

---

**2. 减少重定向**

比如：

```
http://example.com -> https://example.com -> https://www.example.com
```

每次重定向都多一次网络往返。

尽量直接返回最终地址。

---

**3. 使用 CDN**

静态资源放 CDN：

```
<script src="https://cdn.example.com/app.js"></script>
```

CDN 节点离用户近，延迟低。

---

**六、监控和分析**

**1. 使用 Lighthouse**

Chrome DevTools 里自带，可以分析：

```
Performance
Accessibility
Best Practices
SEO
```

给出优化建议和评分。

---

**2. Performance API**

```
const timing = performance.timing

const dns = timing.domainLookupEnd - timing.domainLookupStart
const tcp = timing.connectEnd - timing.connectStart
const ttfb = timing.responseStart - timing.requestStart
const domReady = timing.domContentLoadedEventEnd - timing.fetchStart
const load = timing.loadEventEnd - timing.fetchStart

console.log({ dns, tcp, ttfb, domReady, load })
```

可以上报到监控平台，分析真实用户性能数据。

---

**3. 分析打包体积**

Webpack 可以用：

```
npm install webpack-bundle-analyzer
```

Vite 可以用：

```
npm install rollup-plugin-visualizer
```

生成可视化报告，看哪个包体积大，针对性优化。

---

**面试版回答**

可以这样说：

> Web 优化主要从几个方面入手。加载优化包括压缩资源、懒加载图片、使用 CDN、合理配置缓存策略、preload 关键资源。渲染优化包括避免频繁读写布局、用 transform 和 opacity 做动画、长列表用虚拟滚动、大计算任务用 Web Worker。代码层面做 Tree Shaking、代码分割、防抖节流、避免内存泄漏。构建优化包括生产环境压缩、合理分包、开启 gzip。网络层可以用 HTTP/2、减少重定向、静态资源上 CDN。
> 
> 具体做优化时，通常先用 Lighthouse 或 Performance API 分析性能瓶颈，然后针对性优化，比如首屏慢就优化关键路径，包体积大就做代码分割和 Tree Shaking，列表卡顿就用虚拟列表。优化后再通过监控平台跟踪真实用户数据，形成闭环。


# PWA 是什么

PWA 全称是：

```
Progressive Web App
```

中文一般叫：

```
渐进式 Web 应用
```

它不是某一个框架，而是一组 Web 技术和设计理念。

简单说：

> PWA 是让普通网页拥有接近原生 App 体验的一种方案，比如可以离线访问、添加到桌面、缓存资源、消息推送等。

---

**PWA 主要由什么组成**

PWA 常见核心能力有三个：

```
HTTPS
Web App Manifest
Service Worker
```

---

**1. HTTPS**

PWA 通常要求运行在 HTTPS 下。

原因是 PWA 能力比较敏感，比如缓存请求、拦截网络、推送通知，所以浏览器要求安全上下文。

本地开发的 `localhost` 一般可以例外。

---

**2. Web App Manifest**

Manifest 是一个配置文件，通常叫：

```
manifest.json
```

它描述这个 Web 应用像 App 一样安装到桌面时的表现。

比如：

```
{
  "name": "My App",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#409eff",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

HTML 中引入：

```
<link rel="manifest" href="/manifest.json">
```

它能控制：

- 应用名称
- 桌面图标
- 启动页
- 主题色
- 是否像独立 App 一样打开

比如：

```
"display": "standalone"
```

表示打开时更像原生 App，不显示浏览器地址栏。

---

**3. Service Worker**

Service Worker 是 PWA 的核心。

它是运行在浏览器后台的 JS 脚本，和页面主线程分离。

它可以拦截网络请求：

```
self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request).then(cacheResponse => {
      return cacheResponse || fetch(event.request)
    })
  )
})
```

这段代码意思是：

> 请求资源时，先看缓存里有没有，有就用缓存，没有再走网络。

页面注册 Service Worker：

```
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js')
}
```

Service Worker 常用于：

- 静态资源缓存
- 离线访问
- 请求代理
- 后台同步
- 消息推送

---

**PWA 有什么作用**

**1. 离线访问**

传统网页断网后就打不开。

PWA 可以把核心资源缓存起来：

```
index.html
app.js
style.css
logo.png
```

断网时仍然可以打开页面。

比如文档类、工具类、后台系统的部分页面，就可以做到离线可用。

---

**2. 提升加载速度**

通过 Service Worker 缓存静态资源。

第一次访问：

```
从服务器下载资源并缓存
```

第二次访问：

```
直接从本地缓存读取
```

速度会明显提升。

这对弱网环境很有帮助。

---

**3. 添加到桌面**

通过 Manifest，用户可以把网页添加到手机桌面或电脑应用列表。

打开时像 App 一样：

```
有图标
有启动页
可以全屏 / 独立窗口
```

不一定需要去应用商店下载安装。

---

**4. 消息推送**

PWA 可以结合 Push API 做消息推送。

比如：

```
订单提醒
聊天消息
活动通知
待办提醒
```

即使用户没有打开页面，也可能收到通知。

不过不同浏览器和系统支持程度不完全一样。

---

**5. 更接近原生 App 的体验**

PWA 可以让 Web 应用具备：

```
离线能力
桌面入口
启动页
全屏体验
推送通知
后台同步
```

所以它介于普通网页和原生 App 之间。

---

**PWA 适合什么场景**

比较适合：

- 新闻资讯
- 文档站
- 电商
- 工具类网站
- 低频使用但希望保留桌面入口的应用
- 弱网环境下需要更好体验的业务
- 内部系统的离线缓存部分能力

不一定适合：

- 强依赖原生能力的复杂 App
- 对系统权限要求非常高的应用
- 对 iOS 推送、后台能力要求特别强的场景

---

**PWA 和普通网页的区别**

|对比项|普通网页|PWA|
|---|---|---|
|离线访问|通常不支持|可以支持|
|添加桌面|体验有限|更像 App|
|缓存控制|主要靠 HTTP 缓存|可用 Service Worker 精细控制|
|消息推送|通常不支持|可支持|
|加载速度|依赖网络|可读本地缓存|
|安装成本|打开 URL|可添加到桌面，无需应用商店|

---

**Service Worker 的生命周期**

这个面试可能会追问。

大概有几个阶段：

```
register
install
activate
fetch
```

示例：

```
self.addEventListener('install', event => {
  console.log('install')
})

self.addEventListener('activate', event => {
  console.log('activate')
})

self.addEventListener('fetch', event => {
  console.log('fetch', event.request.url)
})
```

含义：

- `register`：页面注册 sw
- `install`：安装阶段，通常缓存静态资源
- `activate`：激活阶段，通常清理旧缓存
- `fetch`：拦截页面请求

---

**PWA 的缺点**

也要知道，不然回答会显得太理想化。

常见问题：

- 浏览器和系统支持程度不一致，尤其移动端差异明显
- Service Worker 缓存策略写不好，可能导致用户一直看到旧版本
- 调试成本比普通网页高
- 推送通知权限容易被用户拒绝
- 不能完全替代原生 App
- 对 HTTPS 有要求

尤其是缓存更新问题很常见。

比如你缓存了旧的 `app.js`，如果没有设计好更新策略，用户可能上线后还在用旧代码。

---

**面试版回答**

可以这样说：

> PWA 是 Progressive Web App，渐进式 Web 应用。它不是某个框架，而是一组让 Web 应用接近原生 App 体验的技术方案，核心包括 HTTPS、Web App Manifest 和 Service Worker。Manifest 用来描述应用名称、图标、启动方式等，让网页可以添加到桌面；Service Worker 可以运行在浏览器后台，拦截请求并管理缓存，从而实现离线访问、资源缓存、消息推送等能力。
> 
> PWA 的主要作用是提升 Web 应用体验，比如弱网或离线环境下仍然可以访问核心页面，通过缓存提升二次加载速度，也可以添加到桌面，像 App 一样启动。它适合资讯、文档、工具、电商等场景。但它也有缺点，比如兼容性不完全一致，Service Worker 缓存策略复杂，处理不好可能导致用户拿到旧资源，所以实际项目中要设计好缓存更新机制。

HTTP 状态码按首位数字分 5 类，面试里重点掌握常见的这些就够了。

**1xx：信息响应**

表示请求已收到，继续处理。平时业务里不常直接接触。

`100 Continue`  
客户端可以继续发送请求体。常见于大请求体发送前，客户端先询问服务端是否愿意接收。

`101 Switching Protocols`  
协议切换，比如 HTTP 升级到 WebSocket 时可能用到。

---

**2xx：成功**

表示请求被成功接收、理解、处理。

`200 OK`  
请求成功，最常见。

```
GET /user/1 -> 200 OK
```

`201 Created`  
资源创建成功，常见于 POST 新增数据。

```
POST /users -> 201 Created
```

`204 No Content`  
请求成功，但响应体为空。常见于删除成功或更新成功但不返回内容。

```
DELETE /users/1 -> 204 No Content
```

`206 Partial Content`  
部分内容。常见于视频播放、断点续传、Range 请求。

---

**3xx：重定向**

表示需要进一步操作才能完成请求。

`301 Moved Permanently`  
永久重定向。浏览器和搜索引擎会记住新地址。

```
http://example.com -> https://example.com
```

`302 Found`  
临时重定向。旧地址以后还可能继续使用。

`304 Not Modified`  
协商缓存命中。浏览器发送条件请求，服务端告诉浏览器资源没变，可以继续用本地缓存。

常见请求头：

```
If-None-Match
If-Modified-Since
```

响应：

```
304 Not Modified
```

注意：

> `304` 不是错误，是缓存命中。

`307 Temporary Redirect`  
临时重定向，和 `302` 类似，但会保持原请求方法和请求体。

`308 Permanent Redirect`  
永久重定向，和 `301` 类似，但会保持原请求方法和请求体。

---

**4xx：客户端错误**

表示请求有问题，责任主要在客户端。

`400 Bad Request`  
请求参数格式错误，服务端无法理解。

比如 JSON 格式不对、参数类型不对。

`401 Unauthorized`  
未认证。通常是没有登录，或者 token 无效。

```
你是谁？请先登录。
```

`403 Forbidden`  
已认证，但没有权限。

```
我知道你是谁，但你不能访问这个资源。
```

`401` 和 `403` 的区别很重要：

```
401：没登录 / 身份无效
403：登录了，但没权限
```

`404 Not Found`  
资源不存在。比如访问了不存在的接口或页面。

`405 Method Not Allowed`  
请求方法不允许。

比如接口只支持 `GET`，你用了 `POST`。

`408 Request Timeout`  
请求超时，客户端请求发送太慢或服务端等待太久。

`409 Conflict`  
资源冲突。比如用户名已存在、版本冲突、重复提交。

`413 Payload Too Large`  
请求体太大。比如上传文件超过限制。

`415 Unsupported Media Type`  
请求体类型不支持。

比如服务端要求：

```
Content-Type: application/json
```

你发了：

```
Content-Type: text/plain
```

`429 Too Many Requests`  
请求太频繁，被限流。

常见于接口防刷、登录验证码、API 网关限流。

---

**5xx：服务端错误**

表示服务端处理请求时出错。

`500 Internal Server Error`  
服务端内部错误，最常见的服务端异常。

`501 Not Implemented`  
服务器不支持当前请求所需功能。

`502 Bad Gateway`  
网关错误。常见于 Nginx 作为代理时，上游服务异常返回了错误响应。

比如：

```
Nginx -> 后端服务
后端服务返回异常
Nginx 返回 502
```

`503 Service Unavailable`  
服务暂时不可用。比如服务器过载、维护中、服务没启动。

`504 Gateway Timeout`  
网关超时。比如 Nginx 等后端服务响应太久，超时了。

```
Nginx 等后端接口超时 -> 504
```

---

**重点区别**

**301 和 302**

```
301：永久重定向
302：临时重定向
```

301 适合：

```
旧域名永久迁移到新域名
HTTP 永久升级 HTTPS
```

302 适合：

```
临时活动页跳转
登录后临时跳转
```

---

**401 和 403**

```
401：未认证
403：无权限
```

比如：

```
没带 token -> 401
token 有效但不是管理员 -> 403
```

---

**502 和 504**

```
502：网关收到上游错误响应
504：网关等上游响应超时
```

比如：

```
后端服务挂了 / 返回异常 -> 502
后端接口太慢，网关等超时 -> 504
```

---

**面试版回答**

可以这样说：

> HTTP 状态码分为五类：`1xx` 表示信息响应，`2xx` 表示成功，`3xx` 表示重定向，`4xx` 表示客户端错误，`5xx` 表示服务端错误。常见的有 `200` 请求成功，`201` 创建成功，`204` 成功但无响应体；`301` 永久重定向，`302` 临时重定向，`304` 协商缓存命中；`400` 请求参数错误，`401` 未认证，`403` 无权限，`404` 资源不存在，`405` 方法不允许，`429` 请求过于频繁；`500` 服务端内部错误，`502` 网关错误，`503` 服务不可用，`504` 网关超时。
> 
> 面试里常问的区别是：`401` 是没登录或身份无效，`403` 是登录了但没权限；`301` 是永久重定向，`302` 是临时重定向；`502` 是网关收到了上游的错误响应，`504` 是网关等待上游服务超时。