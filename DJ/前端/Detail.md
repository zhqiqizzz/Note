
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

# HTTP 状态码
按首位数字分 5 类，面试里重点掌握常见的这些就够了。

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

# HTTP 缓存机制
主要分两类：

```
强缓存
协商缓存
```

浏览器请求资源时，会先看强缓存；强缓存没命中，再走协商缓存。

---

**一、强缓存**

强缓存的意思是：

> 在缓存有效期内，浏览器直接使用本地缓存，不向服务器发送请求。

常见响应头：

```
Cache-Control: max-age=3600
```

表示资源在 3600 秒内有效。

浏览器下次请求这个资源时，如果还没过期，就直接从缓存读取。

常见表现是 DevTools 里看到：

```
from memory cache
from disk cache
```

或者状态码显示：

```
200 (from cache)
```

---

**强缓存常见字段**

**1. Cache-Control**

HTTP/1.1 常用字段，优先级高。

```
Cache-Control: max-age=3600
```

表示缓存 3600 秒。

常见值：

```
Cache-Control: no-cache
```

注意这个名字容易误解。

`no-cache` 不是不缓存，而是：

> 可以缓存，但每次使用前必须向服务器验证是否过期。

也就是会走协商缓存。

```
Cache-Control: no-store
```

表示：

> 完全不缓存。

请求和响应都不应该存储。

```
Cache-Control: public
```

表示可以被浏览器、CDN 等中间缓存缓存。

```
Cache-Control: private
```

表示只能被用户浏览器缓存，不能被共享缓存，比如 CDN 缓存。

---

**2. Expires**

HTTP/1.0 字段。

```
Expires: Wed, 15 Jun 2026 10:00:00 GMT
```

表示资源在这个时间点前有效。

缺点是依赖客户端本地时间，如果用户电脑时间不准，可能出问题。

所以现在更推荐：

```
Cache-Control: max-age=3600
```

如果 `Cache-Control` 和 `Expires` 同时存在，优先使用 `Cache-Control`。

---

**二、协商缓存**

协商缓存的意思是：

> 浏览器本地有缓存，但不确定能不能继续用，于是向服务器询问资源有没有变化。

如果资源没变，服务器返回：

```
304 Not Modified
```

浏览器继续使用本地缓存。

如果资源变了，服务器返回新的资源：

```
200 OK
```

并带上新的响应内容。

---

**协商缓存常见字段**

**1. Last-Modified / If-Modified-Since**

服务器第一次返回资源时带：

```
Last-Modified: Wed, 15 Jun 2026 09:00:00 GMT
```

浏览器下次请求时带：

```
If-Modified-Since: Wed, 15 Jun 2026 09:00:00 GMT
```

服务器比较资源最后修改时间。

如果没变：

```
304 Not Modified
```

如果变了：

```
200 OK
```

缺点：

- 精度通常是秒级，1 秒内多次修改可能识别不到
- 有些文件内容没变，但修改时间变了，也会导致缓存失效

---

**2. ETag / If-None-Match**

服务器第一次返回资源时带：

```
ETag: "abc123"
```

浏览器下次请求时带：

```
If-None-Match: "abc123"
```

服务器根据 ETag 判断资源内容是否变化。

如果没变：

```
304 Not Modified
```

如果变了：

```
200 OK
```

ETag 通常比 Last-Modified 更精确。

如果两组字段同时存在，通常优先使用：

```
ETag / If-None-Match
```

---

**三、完整缓存流程**

浏览器请求一个资源时，大概是：

```
1. 先看是否有缓存
2. 如果没有缓存，直接请求服务器
3. 如果有缓存，检查强缓存是否有效
4. 强缓存有效，直接用本地缓存
5. 强缓存失效，发请求走协商缓存
6. 服务器判断资源是否变化
7. 没变，返回 304，浏览器用本地缓存
8. 变了，返回 200 和新资源，浏览器更新缓存
```

---

**四、实际项目怎么配置**

静态资源，比如：

```
app.8f3a2c.js
style.a9c331.css
```

文件名带 hash，内容变了文件名也变。

适合配置强缓存：

```
Cache-Control: max-age=31536000, immutable
```

HTML 文件不适合强缓存太久，因为它负责引用最新 JS / CSS。

通常配置：

```
Cache-Control: no-cache
```

让浏览器每次访问 HTML 时都向服务器确认有没有更新。

常见策略：

```
HTML：协商缓存
JS / CSS / 图片等带 hash 静态资源：强缓存
```

这样上线新版本时：

```
用户请求 index.html
-> 发现 HTML 更新
-> HTML 引用新的 app.xxxxx.js
-> 浏览器下载新的 JS
```

旧 JS 因为文件名不同，不会影响新版本。

---

**五、面试版回答**

可以这样说：

> HTTP 缓存主要分为强缓存和协商缓存。浏览器请求资源时会先判断强缓存是否命中，如果命中，就直接从本地缓存读取，不会发送请求。强缓存主要通过 `Cache-Control` 和 `Expires` 控制，其中 `Cache-Control` 优先级更高，比如 `max-age` 表示缓存有效时间。
> 
> 如果强缓存失效，就会走协商缓存。浏览器会带上 `If-None-Match` 或 `If-Modified-Since` 去询问服务器资源是否变化。服务器如果判断资源没变，就返回 `304 Not Modified`，浏览器继续使用本地缓存；如果资源变了，就返回 `200` 和新的资源。协商缓存主要通过 `ETag / If-None-Match` 和 `Last-Modified / If-Modified-Since` 实现，通常 `ETag` 优先级更高。
> 
> 实际项目里，一般 HTML 使用协商缓存，JS、CSS、图片等带 hash 的静态资源使用长期强缓存。这样既能利用缓存提升性能，也能保证发布新版本后用户能拿到最新资源。


# HTTP1/2/3核心区别

可以先记一句主线：

> HTTP/1.1 主要解决了连接复用；HTTP/2 主要解决了多路复用和头部压缩；HTTP/3 把底层从 TCP 换成了基于 UDP 的 QUIC，进一步解决 TCP 队头阻塞和连接迁移问题。

---

**HTTP/1.1**

HTTP/1.1 是现在仍然很常见的版本。

它相比 HTTP/1.0，重要改进是：

```
持久连接
管道化
缓存控制增强
Host 头
分块传输
```

**1. 持久连接**

HTTP/1.0 默认一个请求一个 TCP 连接：

```
请求 HTML -> 建 TCP -> 响应 -> 断开
请求 CSS  -> 建 TCP -> 响应 -> 断开
请求 JS   -> 建 TCP -> 响应 -> 断开
```

成本很高。

HTTP/1.1 默认开启长连接：

```
Connection: keep-alive
```

一个 TCP 连接可以复用多个请求：

```
TCP 连接
  -> 请求 HTML
  -> 请求 CSS
  -> 请求 JS
```

减少了频繁建连开销。

---

**2. 队头阻塞**

HTTP/1.1 虽然支持复用 TCP 连接，但同一个连接上的请求响应基本还是按顺序处理。

如果前一个请求慢了，后面的响应就会被阻塞。

比如：

```
请求 A：大图片，很慢
请求 B：小 JS，很快
请求 C：小 CSS，很快
```

如果它们在同一个连接上，B、C 可能要等 A。

这叫：

```
HTTP 层队头阻塞
```

所以浏览器通常会对同一域名开多个 TCP 连接来缓解，比如 6 个左右。

---

**HTTP/2**

HTTP/2 相比 HTTP/1.1，核心改进是：

```
二进制分帧
多路复用
头部压缩
服务器推送
请求优先级
```

---

**1. 二进制分帧**

HTTP/1.1 报文是文本格式：

```
GET /index.html HTTP/1.1
Host: example.com
```

HTTP/2 把数据拆成二进制帧：

```
Headers Frame
Data Frame
```

一个请求或响应被拆成多个帧传输。

这为多路复用打基础。

---

**2. 多路复用**

HTTP/2 可以在一个 TCP 连接上同时传输多个请求和响应。

比如：

```
一个 TCP 连接
  -> 请求 A 的帧
  -> 请求 B 的帧
  -> 请求 C 的帧
  -> 响应 B 的帧
  -> 响应 C 的帧
  -> 响应 A 的帧
```

这些帧可以交错传输。

所以不需要像 HTTP/1.1 那样开很多连接。

优点：

```
减少 TCP 连接数量
提高并发能力
减少 HTTP 层队头阻塞
```

---

**3. 头部压缩**

HTTP 请求头里有很多重复内容：

```
Cookie
User-Agent
Accept
Authorization
```

HTTP/1.1 每次都要重复传输。

HTTP/2 使用 HPACK 压缩头部，减少请求头体积。

这对大量小请求很有帮助。

---

**4. HTTP/2 仍然有 TCP 队头阻塞**

注意，HTTP/2 解决的是：

```
HTTP 层面的队头阻塞
```

但它还是基于 TCP。

TCP 要保证字节流有序可靠。

如果一个 TCP 包丢了，后面的包即使已经到了，也要等丢失的包重传后才能交给上层。

所以 HTTP/2 仍然可能受到：

```
TCP 层队头阻塞
```

影响。

这就是 HTTP/3 要解决的问题。

---

**HTTP/3**

HTTP/3 最大变化是：

```
不再基于 TCP，而是基于 QUIC
```

QUIC 是基于 UDP 实现的传输协议。

可以理解为：

```
HTTP/1.1 -> TCP
HTTP/2   -> TCP
HTTP/3   -> QUIC -> UDP
```

---

**1. QUIC 解决 TCP 队头阻塞**

HTTP/3 的 QUIC 内部支持多路复用。

不同请求的数据流相互独立。

如果某个流丢包，只影响这个流，不会阻塞其他流。

比如：

```
流 A 丢包
流 B 可以继续处理
流 C 可以继续处理
```

这就解决了 HTTP/2 在 TCP 层的队头阻塞问题。

---

**2. 更快的连接建立**

HTTP/2 over HTTPS 通常需要：

```
TCP 握手
TLS 握手
HTTP 请求
```

HTTP/3 的 QUIC 内置 TLS 1.3，可以减少握手往返。

对于已经访问过的网站，还可以支持 0-RTT，连接恢复更快。

---

**3. 连接迁移**

TCP 连接由四元组标识：

```
源 IP
源端口
目标 IP
目标端口
```

如果你从 Wi-Fi 切换到 4G，IP 变了，TCP 连接通常要断开重连。

QUIC 使用 Connection ID，不强依赖 IP 和端口。

所以网络切换时，可以保持连接迁移。

这对移动端体验很重要。

---

**对比表**

|对比项|HTTP/1.1|HTTP/2|HTTP/3|
|---|---|---|---|
|底层协议|TCP|TCP|QUIC over UDP|
|数据格式|文本|二进制帧|二进制帧|
|连接复用|支持长连接|一个连接多路复用|多路复用，流更独立|
|队头阻塞|有 HTTP 层队头阻塞|解决 HTTP 层，但仍有 TCP 层阻塞|解决 TCP 队头阻塞|
|头部压缩|无专门压缩|HPACK|QPACK|
|握手速度|TCP + TLS|TCP + TLS|QUIC 内置 TLS 1.3，更快|
|连接迁移|不支持|不支持|支持|
|典型优势|简单成熟|并发更好、头部更小|弱网和移动网络表现更好|

---

**实际项目里的影响**

HTTP/1.1 时代经常做：

```
合并 JS/CSS
雪碧图
域名分片
减少请求数
```

因为连接并发有限，请求多会慢。

HTTP/2 以后：

```
多路复用更强
不一定要过度合并小文件
域名分片反而可能破坏连接复用
```

但代码分包仍然重要，因为 JS 体积大依然会影响解析和执行。

HTTP/3 对用户感知更明显的场景：

```
弱网
高延迟
移动网络切换
大量并发请求
视频/直播等场景
```

---

**面试版回答**

可以这样说：

> HTTP/1.1 默认支持持久连接，可以复用 TCP 连接，但同一个连接上的请求响应仍然容易出现队头阻塞，所以浏览器通常会对同一域名建立多个连接来提高并发。
> 
> HTTP/2 引入了二进制分帧、多路复用、头部压缩等能力，可以在一个 TCP 连接上并发传输多个请求和响应，解决了 HTTP 层面的队头阻塞，也减少了连接数量和请求头体积。但因为 HTTP/2 仍然基于 TCP，如果 TCP 层发生丢包，还是会阻塞后续数据交付。
> 
> HTTP/3 则把底层从 TCP 换成了基于 UDP 的 QUIC。QUIC 内置 TLS 1.3，连接建立更快，并且支持多路复用和连接迁移。它可以避免 TCP 层的队头阻塞，一个流丢包不会影响其他流，所以在弱网和移动网络切换场景下表现更好。

# HTTPS 的加密方式是什么？为什么会是安全的？

HTTPS 不是单纯一种加密算法，它是：

```
HTTP + TLS
```

也就是在 HTTP 和 TCP 之间加了一层 TLS 安全层。

HTTPS 的安全主要靠三件事：

```
加密传输
身份认证
完整性校验
```

---

**一、HTTPS 用了什么加密方式**

HTTPS 同时用了两类加密：

```
非对称加密
对称加密
```

再配合：

```
数字证书
数字签名
哈希摘要
消息认证码
```

---

**1. 非对称加密**

非对称加密有一对密钥：

```
公钥
私钥
```

公钥可以公开，私钥只有服务器自己保存。

特点：

```
公钥加密的数据，只能用私钥解密
私钥签名的数据，可以用公钥验证
```

HTTPS 握手阶段会用到非对称加密或密钥交换。

但非对称加密比较慢，所以不会用它来加密所有 HTTP 数据。

---

**2. 对称加密**

对称加密只有一个密钥：

```
客户端和服务端使用同一个会话密钥
```

它的特点是速度快。

HTTPS 真正传输业务数据时，主要用对称加密，比如 AES、ChaCha20。

也就是说：

```
握手阶段：协商出会话密钥
传输阶段：用会话密钥对 HTTP 内容加密
```

---

**二、HTTPS 大致握手流程**

以简化版 TLS 流程来说：

1. 客户端发起请求，告诉服务器自己支持哪些 TLS 版本和加密套件。

```
ClientHello
```

2. 服务器返回选择的 TLS 版本、加密套件，以及数字证书。

```
ServerHello + Certificate
```

3. 客户端验证服务器证书是否合法。

验证内容包括：

```
证书是否过期
证书域名是否匹配
证书是否由可信 CA 签发
证书签名是否正确
```

4. 验证通过后，客户端和服务器通过密钥交换算法协商出会话密钥。
    
5. 后续 HTTP 请求和响应都使用这个会话密钥进行对称加密。
    

---

**三、证书是干什么的**

证书解决的问题是：

> 我怎么知道对方真的是我要访问的网站，而不是中间人？

比如你访问：

```
https://bank.com
```

服务器会发一个证书给浏览器。

证书里面包含：

```
域名
公钥
证书有效期
签发机构 CA
签名
```

浏览器会用系统内置的可信 CA 根证书去验证这个证书。

如果证书可信，浏览器才认为：

```
这个公钥确实属于 bank.com
```

否则中间人可以伪造一个公钥骗你。

---

**四、为什么 HTTPS 能防中间人攻击**

假设中间人在你和服务器之间拦截通信。

如果没有证书认证，中间人可以这样：

```
客户端以为自己拿到的是服务器公钥
实际拿到的是中间人的公钥
```

然后中间人就能解密、篡改、再转发。

HTTPS 用 CA 证书体系避免这个问题。

中间人如果伪造证书，浏览器会检查：

```
这个证书是不是可信 CA 签发的？
证书域名是不是当前访问域名？
证书签名能不能验证通过？
```

如果不能通过，浏览器会提示证书错误。

所以 HTTPS 不只是加密，还验证身份。

---

**五、为什么不用非对称加密传全部数据**

因为非对称加密很慢。

如果每个 HTTP 请求、响应都用 RSA 这种方式加密，性能很差。

所以 HTTPS 使用混合加密：

```
用非对称加密 / 密钥交换解决密钥安全协商问题
用对称加密加密真正的数据传输
```

这样既安全，又高效。

---

**六、完整性校验是什么**

HTTPS 还要保证数据没有被篡改。

比如中间人不能把：

```
转账 100 元
```

改成：

```
转账 10000 元
```

TLS 会通过消息认证码或 AEAD 算法保证完整性。

如果数据被篡改，接收方校验不通过，会丢弃连接。

所以 HTTPS 能保证：

```
数据被加密，看不到
数据被篡改，能发现
对方身份，可验证
```

---

**七、TLS 1.2 和 TLS 1.3 简单区别**

现在主流是 TLS 1.2 和 TLS 1.3。

TLS 1.3 更现代：

```
减少握手次数
移除不安全算法
默认使用更安全的密钥交换方式
性能更好
```

你面试一般不用展开太深，但可以提一句：

> 现代 HTTPS 通常基于 TLS 1.2 或 TLS 1.3，TLS 1.3 握手更快，也移除了很多旧的不安全算法。

---

**八、面试版回答**

可以这样说：

> HTTPS 本质是 HTTP 加 TLS。它使用的是混合加密机制：握手阶段通过非对称加密或密钥交换算法安全地协商出会话密钥，之后真正传输 HTTP 数据时使用对称加密，因为对称加密性能更高。
> 
> HTTPS 安全主要体现在三个方面：第一是加密传输，第三方即使抓包也看不到明文内容；第二是身份认证，服务器会提供由 CA 签发的数字证书，浏览器会验证证书是否合法、是否过期、域名是否匹配，从而确认访问的确实是目标服务器；第三是完整性校验，TLS 会校验数据是否被篡改，如果中间人修改了数据，接收方可以发现。
> 
> 所以 HTTPS 不是只靠加密，而是通过加密、证书认证和完整性校验一起保证通信安全。



# SSR 渲染、CSR渲染、SSG 渲染、同构渲染的区别是什么？

**先给一句总览**

这几个概念主要区别在于：

```
HTML 是在哪里、什么时候生成的
```

- **CSR**：HTML 主要在浏览器里由 JS 生成
- **SSR**：HTML 在服务器收到请求时实时生成
- **SSG**：HTML 在构建阶段提前生成
- **同构渲染**：同一套代码既能在服务端渲染，也能在客户端接管

---

**CSR：客户端渲染**

CSR 全称：

```
Client Side Rendering
```

也就是客户端渲染。

常见 Vue SPA、React SPA 默认就是 CSR。

用户访问页面时，服务端通常返回一个很空的 HTML：

```
<div id="app"></div>
<script src="/assets/app.js"></script>
```

浏览器下载 JS 后，JS 执行，再请求接口，最后生成页面内容。

流程：

```
浏览器请求页面
  ↓
服务器返回空 HTML + JS
  ↓
浏览器下载并执行 JS
  ↓
JS 请求接口数据
  ↓
前端生成 DOM
  ↓
页面可见、可交互
```

优点：

- 前后端分离清晰
- 页面切换体验好
- 服务器压力较小
- 适合后台管理系统、交互型应用

缺点：

- 首屏可能慢
- SEO 不友好
- 低端设备上 JS 执行压力大
- 白屏时间可能较长

---

**SSR：服务端渲染**

SSR 全称：

```
Server Side Rendering
```

也就是服务端渲染。

用户请求页面时，服务端根据当前路由和数据，实时生成完整 HTML 返回给浏览器。

比如服务端返回：

```
<div id="app">
  <h1>商品详情</h1>
  <p>价格：99 元</p>
</div>
<script src="/assets/app.js"></script>
```

流程：

```
浏览器请求页面
  ↓
服务器根据路由请求数据
  ↓
服务器生成完整 HTML
  ↓
浏览器先展示 HTML
  ↓
下载 JS
  ↓
客户端 hydrate，绑定事件
  ↓
页面可交互
```

这里有个关键词：

```
hydrate / 水合
```

意思是：

> 服务端已经把 HTML 渲染出来了，客户端 JS 加载后，不是重新生成一遍 DOM，而是在已有 HTML 上绑定事件、恢复状态，让页面变成可交互。

优点：

- 首屏内容更快可见
- SEO 更友好
- 弱网下能更早看到内容

缺点：

- 服务端压力更大
- 架构复杂
- 要处理服务端和客户端环境差异
- 请求量大时需要缓存优化

适合：

- 官网
- 内容站
- 新闻站
- 电商详情页
- 需要 SEO 和首屏性能的页面

---

**SSG：静态站点生成**

SSG 全称：

```
Static Site Generation
```

也就是静态生成。

它和 SSR 的区别是：

```
SSR：请求时生成 HTML
SSG：构建时提前生成 HTML
```

比如博客有 100 篇文章，构建时提前生成：

```
dist/post/1.html
dist/post/2.html
dist/post/3.html
...
```

用户访问时，服务器或 CDN 直接返回静态 HTML。

流程：

```
构建阶段获取数据
  ↓
提前生成 HTML
  ↓
部署到 CDN
  ↓
用户请求页面
  ↓
直接返回静态 HTML
```

优点：

- 访问速度快
- 部署简单
- CDN 友好
- 服务器压力小
- SEO 好

缺点：

- 数据更新不够实时
- 内容变化后通常要重新构建
- 页面数量特别大时构建时间长

适合：

- 博客
- 文档站
- 营销页
- 官网
- 内容更新不频繁的页面

---

**同构渲染**

同构渲染也常叫：

```
Isomorphic Rendering
Universal Rendering
```

意思是：

> 同一套代码既可以在服务端运行，也可以在客户端运行。

典型场景就是 SSR 框架：

```
Next.js
Nuxt
```

以 Vue / React 为例，同一个页面组件：

```
<template>
  <div>{{ title }}</div>
</template>
```

服务端可以先执行它，生成 HTML。

客户端也可以加载同一套组件代码，进行 hydrate，然后接管页面交互。

所以同构渲染更像是 SSR 的一种工程实现方式。

它解决的是：

```
服务端渲染一份 HTML
客户端继续使用同一套代码接管页面
```

注意：

> SSR 强调“服务端生成 HTML”；同构强调“服务端和客户端复用同一套代码”。

---

**四者对比**

|方式|HTML 生成位置|HTML 生成时机|首屏|SEO|服务器压力|适合场景|
|---|---|---|---|---|---|---|
|CSR|浏览器|用户访问后 JS 执行|较慢|较差|较小|后台系统、复杂交互应用|
|SSR|服务端|每次请求时|较快|好|较大|电商、新闻、内容详情页|
|SSG|构建阶段|构建时提前生成|很快|好|很小|博客、文档、官网|
|同构|服务端 + 客户端|服务端先渲染，客户端接管|快|好|看实现|Nuxt、Next 这类 SSR 应用|

---

**CSR 和 SSR 的关键区别**

CSR 返回的 HTML 通常很空：

```
<div id="app"></div>
```

主要内容靠浏览器 JS 生成。

SSR 返回的 HTML 已经有内容：

```
<div id="app">
  <h1>商品详情</h1>
</div>
```

所以：

```
CSR：先加载 JS，再看到内容
SSR：先看到 HTML 内容，再加载 JS 接管
```

---

**SSR 和 SSG 的关键区别**

SSR：

```
每次请求时实时生成 HTML
```

适合数据变化频繁，且需要 SEO 的页面。

SSG：

```
构建时提前生成 HTML
```

适合内容相对稳定的页面。

比如商品详情：

- 秒杀库存、价格经常变：更适合 SSR 或 CSR 动态请求
- 文档文章、博客内容稳定：更适合 SSG

---

**同构里的水合 Hydration**

这个面试很容易追问。

SSR 返回 HTML 后，用户可以先看到页面内容。

但这时候按钮还没有绑定前端事件。

比如：

```
<button>加入购物车</button>
```

JS 加载完成后，框架会在已有 DOM 上绑定事件：

```
button.addEventListener('click', addToCart)
```

这个过程就是水合。

如果服务端生成的 HTML 和客户端第一次渲染的结构不一致，就可能出现 hydration mismatch。

比如服务端渲染：

```
<div>登录</div>
```

客户端首次渲染：

```
<div>用户名：Tom</div>
```

就可能产生水合不匹配。

---

**面试版回答**

可以这样说：

> CSR 是客户端渲染，服务端通常只返回一个空的 HTML 容器和 JS，页面内容由浏览器下载并执行 JS 后生成，适合后台系统和交互型应用，但首屏和 SEO 较弱。
> 
> SSR 是服务端渲染，用户请求页面时，服务端实时获取数据并生成完整 HTML 返回，浏览器可以更快看到首屏内容，SEO 也更好，但服务器压力和工程复杂度更高。
> 
> SSG 是静态站点生成，它在构建阶段提前把页面生成成 HTML，访问时直接返回静态文件，所以速度快、CDN 友好、SEO 好，但数据更新不够实时，适合博客、文档、官网这类内容稳定的场景。
> 
> 同构渲染强调同一套代码同时运行在服务端和客户端。服务端先用这套代码生成 HTML，客户端加载 JS 后再进行 hydration，在已有 HTML 上绑定事件并接管交互。SSR 更强调渲染发生在服务端，同构更强调代码复用和客户端接管。

# CSR和SSR具体流程
**CSR 完整流程**

假设你访问一个 Vue SPA：

```
https://app.example.com
```

---

**1. 浏览器请求 HTML**

浏览器发起请求：

```
GET / HTTP/1.1
Host: app.example.com
```

服务器返回一个很轻的 HTML：

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>App</title>
  <link rel="icon" href="/favicon.ico">
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/assets/app.js"></script>
</body>
</html>
```

注意这个 HTML 几乎没有实际业务内容。

只有：

```
<div id="app"></div>
```

页面主体是空的。

---

**2. 浏览器解析 HTML**

浏览器收到 HTML 后开始解析。

遇到：

```
<div id="app"></div>
```

会先创建一个空 DOM 节点。

然后遇到：

```
<script type="module" src="/assets/app.js"></script>
```

浏览器会：

- 发起请求下载 `app.js`
- 如果有缓存，可能从缓存读
- 如果是 `type="module"`，默认行为类似 `defer`，不会阻塞 HTML 解析

这时页面上看到的只是一个空白页面，或者 loading 占位符。

---

**3. 下载 JS**

浏览器请求：

```
GET /assets/app.js
```

服务器返回打包后的 JS，可能几百 KB 到几 MB。

这个文件通常包括：

```
Vue / React 框架代码
业务组件代码
路由代码
状态管理
工具库
```

如果是生产环境，通常还会代码分割：

```
app.js：主包
vendor.js：第三方库
chunk-xxx.js：按需加载的模块
```

浏览器下载这些 JS 需要时间。

如果网络慢，下载可能要几秒。

---

**4. 执行 JS**

JS 下载完成后，浏览器开始执行。

执行过程中会：

```
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import store from './store'

const app = createApp(App)
app.use(router)
app.use(store)
app.mount('#app')
```

这时 Vue 开始工作：

- 初始化路由
- 解析当前 URL 路径
- 匹配对应组件
- 如果组件需要数据，发起接口请求

---

**5. 请求接口数据**

通常前端组件会在生命周期里请求接口：

```
async created() {
  const res = await fetch('/api/user')
  this.user = await res.json()
}
```

或者 React：

```
useEffect(() => {
  fetch('/api/user')
    .then(res => res.json())
    .then(data => setUser(data))
}, [])
```

这又是一轮网络请求。

如果接口慢，页面可能继续 loading。

---

**6. 生成 DOM 并渲染**

接口返回后，前端框架根据数据生成 DOM：

```
render() {
  return <div>{user.name}</div>
}
```

浏览器进行布局、绘制、合成，用户才能看到实际内容。

---

**CSR 时间线总结**

```
t0：用户输入 URL
  ↓
t1：浏览器请求 HTML
  ↓
t2：HTML 返回，开始解析（内容为空）
  ↓
t3：浏览器请求 app.js
  ↓
t4：app.js 下载完成
  ↓
t5：浏览器执行 JS，框架初始化
  ↓
t6：前端发起接口请求
  ↓
t7：接口返回数据
  ↓
t8：前端生成 DOM
  ↓
t9：浏览器渲染，用户看到内容
  ↓
t10：JS 继续执行，页面可交互
```

关键问题是：

> 用户要等到 t9 才能看到实际内容。

在这之前看到的可能是：

```
白屏
loading 动画
骨架屏
```

---

**SSR 完整流程**

现在看 SSR，假设用 Nuxt / Next.js。

访问：

```
https://app.example.com/product/123
```

---

**1. 浏览器请求 HTML**

```
GET /product/123 HTTP/1.1
Host: app.example.com
```

---

**2. 服务端接收请求**

Node 服务器收到请求后：

- 解析路由，知道用户访问的是商品详情页
- 执行对应页面组件的服务端逻辑

比如 Nuxt 里：

```
export default {
  async asyncData({ params }) {
    const res = await fetch(`https://api.example.com/product/${params.id}`)
    const product = await res.json()
    return { product }
  }
}
```

服务端在这里已经请求接口，拿到数据。

---

**3. 服务端生成 HTML**

服务端把 Vue 组件和数据结合，渲染成 HTML 字符串。

类似：

```
const html = renderToString(App, { product })
```

生成的 HTML 大概是：

```
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>商品详情</title>
  <link rel="stylesheet" href="/assets/app.css">
</head>
<body>
  <div id="app">
    <div class="product">
      <h1>iPhone 15 Pro</h1>
      <p class="price">¥7999</p>
      <button class="btn">加入购物车</button>
    </div>
  </div>

  <script>
    window.__INITIAL_STATE__ = {
      product: {
        name: 'iPhone 15 Pro',
        price: 7999
      }
    }
  </script>

  <script type="module" src="/assets/app.js"></script>
</body>
</html>
```

注意：

- HTML 已经包含实际业务内容
- 服务端把数据注入到 `window.__INITIAL_STATE__`
- 依然引入了客户端 JS

---

**4. 服务器返回 HTML**

服务器把这个完整 HTML 返回给浏览器。

---

**5. 浏览器解析 HTML**

浏览器收到 HTML 后：

```
解析 HTML
  ↓
创建 DOM 树
  ↓
用户已经可以看到内容
```

这时页面上已经有：

```
<h1>iPhone 15 Pro</h1>
<p class="price">¥7999</p>
<button class="btn">加入购物车</button>
```

用户能看到完整内容，但还不能交互。

点击按钮可能没反应，因为事件还没绑定。

---

**6. 下载 JS**

浏览器继续解析到：

```
<script type="module" src="/assets/app.js"></script>
```

开始下载客户端 JS。

这个 JS 和 CSR 的 JS 基本一样，包含 Vue / React 框架和业务代码。

---

**7. 执行 JS 并 hydrate**

JS 下载完成后，浏览器执行。

但这时和 CSR 不同：

CSR 是：

```
createApp(App).mount('#app')
```

直接在空 `<div>` 里创建 DOM。

SSR 是：

```
createApp(App).hydrate('#app')
```

或者 React：

```
hydrateRoot(document.getElementById('app'), <App />)
```

它不会重新创建 DOM，而是：

- 检查服务端渲染的 HTML
- 把 Vue / React 组件实例和已有 DOM 关联起来
- 绑定事件监听器
- 恢复状态

比如：

```
<button class="btn">加入购物车</button>
```

hydrate 后变成：

```
<button class="btn" @click="addToCart">加入购物车</button>
```

这个过程叫 hydration / 水合。

---

**8. 页面可交互**

hydrate 完成后，页面变成可交互状态。

用户点击按钮，事件触发，可以正常操作了。

---

**SSR 时间线总结**

```
t0：用户输入 URL
  ↓
t1：浏览器请求 HTML
  ↓
t2：服务端接收请求
  ↓
t3：服务端请求接口数据
  ↓
t4：服务端生成完整 HTML
  ↓
t5：HTML 返回浏览器
  ↓
t6：浏览器解析 HTML，用户看到内容（但不能交互）
  ↓
t7：浏览器请求 app.js
  ↓
t8：app.js 下载完成
  ↓
t9：浏览器执行 JS，进行 hydrate
  ↓
t10：页面可交互
```

关键区别：

> 用户在 t6 就能看到内容了，而不是等到 JS 执行后。

---

**对比 CSR 和 SSR 的关键节点**

|对比项|CSR|SSR|
|---|---|---|
|HTML 返回时内容|空，只有 `<div id="app"></div>`|完整内容，包括业务数据|
|用户何时看到内容|JS 执行后，接口请求返回后|HTML 解析后立刻可见|
|接口在哪里请求|浏览器|服务端|
|首屏速度|慢，要等 JS + 接口|快，HTML 直接有内容|
|JS 的作用|创建 DOM 并绑定事件|hydrate，复用 DOM 并绑定事件|
|SEO|差，爬虫看到的是空 HTML|好，爬虫能拿到完整 HTML|

---

**hydrate 过程细节**

这个是 SSR 的核心。

服务端渲染出来的 HTML：

```
<div id="app">
  <h1>iPhone 15 Pro</h1>
  <button class="btn">加入购物车</button>
</div>
```

客户端 JS 加载后，会执行类似：

```
const app = createApp(App)
app.hydrate('#app')
```

框架会：

1. 根据服务端注入的初始状态恢复数据：

```
const initialState = window.__INITIAL_STATE__
```

2. 遍历已有 DOM 节点
    
3. 把 Vue / React 组件实例和对应 DOM 节点关联
    
4. 绑定事件：
    

```
button.addEventListener('click', addToCart)
```

5. 激活响应式系统

如果服务端渲染和客户端第一次渲染不一致，就会报错：

```
Hydration mismatch
```

比如服务端生成：

```
<div>登录</div>
```

客户端 hydrate 时发现应该是：

```
<div>用户名：Tom</div>
```

框架会警告或直接用客户端重新渲染。

常见原因：

- 服务端和客户端获取的数据不一致
- 使用了浏览器专属 API，比如 `window.innerWidth`
- 使用了随机数、时间戳等不确定因素

---

**为什么 SSR 还要下载 JS**

你可能会问：

> SSR 既然服务端已经生成了完整 HTML，为什么客户端还要下载 JS？

因为 HTML 只是静态结构，没有交互能力。

比如：

```
<button>加入购物车</button>
```

这只是一个按钮，没有绑定事件。

点击没反应。

客户端 JS 加载后，才会：

```
button.addEventListener('click', addToCart)
```

所以 SSR 的目标是：

```
用户尽快看到内容（通过服务端生成 HTML）
同时保留前端框架的交互能力（通过客户端 hydrate）
```

这就是为什么 SSR 需要同构：

```
同一套代码先在服务端渲染
再在客户端接管
```

---

**SSR 如何处理动态数据**

如果商品价格经常变，SSR 怎么办？

常见策略：

**1. 每次请求都生成最新 HTML**

```
export default {
  async asyncData({ params }) {
    const product = await fetchProduct(params.id)
    return { product }
  }
}
```

每次用户访问，服务端实时请求接口。

缺点是服务端压力大。

**2. 配合缓存**

可以在 Node 服务或 CDN 层做缓存：

```
cache.set(`product:${id}`, html, 60) // 缓存 60 秒
```

用户访问时先看缓存，减轻服务端压力。

**3. 部分 SSR + 部分 CSR**

首屏关键内容走 SSR。

不太重要、变化频繁的部分，客户端再请求：

```
mounted() {
  this.fetchComments()
}
```

比如商品详情走 SSR，评论列表走客户端动态加载。

---

**面试版完整回答**

可以这样说：

> CSR 的流程是浏览器请求 HTML，服务端返回一个几乎空的 HTML 和 script 标签，浏览器解析后开始下载 JS，JS 下载完成后执行框架初始化，前端再发起接口请求获取数据，数据返回后生成 DOM 并渲染，用户才能看到实际内容。所以 CSR 的首屏慢，SEO 差，但前后端分离清晰，页面切换体验好。
> 
> SSR 的流程是浏览器请求 HTML，服务端接收请求后根据路由匹配组件，在服务端执行组件的数据获取逻辑，比如调用接口，拿到数据后把组件和数据一起渲染成完整 HTML 字符串，并把数据序列化注入到 `window.__INITIAL_STATE__`，返回给浏览器。浏览器解析 HTML 后用户就能看到完整内容，但这时还不能交互。接着浏览器下载客户端 JS，JS 执行后进行 hydration，也就是在已有 DOM 上绑定事件、恢复状态、接管交互，不会重新创建 DOM。hydrate 完成后页面可交互。
> 
> 所以 SSR 的关键是服务端先渲染出有内容的 HTML，让用户更快看到首屏，客户端 JS 再通过 hydrate 接管页面，保留前端框架的交互能力。SSR 首屏快、SEO 好，但服务端压力大，需要处理数据获取、HTML 生成、hydration mismatch 等问题。

# 虚拟dom
**虚拟 DOM 是什么**

虚拟 DOM 是 JS 对象，不是真实 DOM 节点。

Vue / React 组件的 render 函数执行后，会生成类似这样的结构：

```
{
  type: 'div',
  props: { class: 'product' },
  children: [
    {
      type: 'h1',
      props: {},
      children: ['iPhone 15 Pro']
    },
    {
      type: 'button',
      props: { class: 'btn' },
      children: ['加入购物车']
    }
  ]
}
```

它只是描述页面应该长什么样的数据结构，存在内存里。

**真实 DOM 是什么**

真实 DOM 是浏览器里实际的节点对象：

```
document.createElement('div')
document.querySelector('.product')
```

是浏览器提供的原生对象，操作它会影响页面视觉。

---

**一、CSR 每一步涉及什么**

```
浏览器请求 HTML
```

```
服务端返回空 HTML
```

```
<div id="app"></div>
<script src="/app.js"></script>
```

```
浏览器解析 HTML
```

这一步浏览器原生工作，**生成真实 DOM**。

此时真实 DOM 里只有：

```
document.body
  └─ div#app（空的）
  └─ script
```

没有业务内容，没有虚拟 DOM，什么都没有。

---

```
下载并执行 app.js
```

JS 里有 Vue / React 框架代码和业务组件。

执行 `createApp(App).mount('#app')` 后：

**第一步：生成虚拟 DOM**

Vue 执行 App 组件的 render 函数，生成虚拟 DOM 树：

```
// 虚拟 DOM，存在内存里
{
  type: 'div',
  props: { class: 'product' },
  children: [
    { type: 'h1', children: ['iPhone 15 Pro'] },
    { type: 'button', children: ['加入购物车'] }
  ]
}
```

**第二步：生成真实 DOM**

Vue 根据虚拟 DOM，调用浏览器 API 创建真实 DOM：

```
const div = document.createElement('div')
div.className = 'product'

const h1 = document.createElement('h1')
h1.textContent = 'iPhone 15 Pro'

const btn = document.createElement('button')
btn.textContent = '加入购物车'

div.appendChild(h1)
div.appendChild(btn)

document.querySelector('#app').appendChild(div)
```

**第三步：挂载到页面**

真实 DOM 插入到 `<div id="app">` 里，浏览器渲染出页面。

---

**CSR 里 DOM 生成总结**

```
解析 HTML          → 浏览器原生生成真实 DOM（只有空 #app）
JS 执行 render     → Vue/React 生成虚拟 DOM
框架 patch/commit  → Vue/React 根据虚拟 DOM 生成真实 DOM
真实 DOM 插入页面  → 浏览器渲染，用户看到内容
```

所以 CSR 里，虚拟 DOM 先生成，再用它指导创建真实 DOM。

---

**二、SSR 每一步涉及什么**

SSR 有两个阶段：**服务端**和**客户端**，要分开看。

---

**服务端阶段**

```
服务端接收请求，执行 Vue/React 组件
```

**第一步：服务端生成虚拟 DOM**

服务端执行 Vue / React 组件的 render 函数，**同样会生成虚拟 DOM**：

```
// 服务端内存里的虚拟 DOM
{
  type: 'div',
  props: { class: 'product' },
  children: [
    { type: 'h1', children: ['iPhone 15 Pro'] },
    { type: 'button', children: ['加入购物车'] }
  ]
}
```

注意：

> 服务端没有浏览器，没有真实 DOM，不能调用 `document.createElement`。

**第二步：虚拟 DOM 序列化成 HTML 字符串**

Vue 的 `renderToString` 方法把虚拟 DOM 转成 HTML 字符串：

```
const html = await renderToString(app)
```

得到：

```
<div class="product" data-v-app="">
  <h1>iPhone 15 Pro</h1>
  <button class="btn">加入购物车</button>
</div>
```

注意：

> 服务端没有生成真实 DOM，只是虚拟 DOM → HTML 字符串。

**第三步：服务端返回完整 HTML**

服务端把这段字符串嵌入 HTML 模板：

```
<div id="app">
  <div class="product">
    <h1>iPhone 15 Pro</h1>
    <button class="btn">加入购物车</button>
  </div>
</div>
<script>
  window.__INITIAL_STATE__ = { product: { name: 'iPhone 15 Pro', price: 7999 } }
</script>
<script src="/app.js"></script>
```

返回给浏览器。

---

**客户端阶段**

```
浏览器收到 HTML，开始解析
```

**第一步：浏览器原生生成真实 DOM**

浏览器解析服务端返回的 HTML，生成真实 DOM 树：

```
document.body
  └─ div#app
       └─ div.product
            ├─ h1（iPhone 15 Pro）
            └─ button（加入购物车）
```

这是浏览器自己原生做的，和框架无关。

用户此时已经能看到内容，因为真实 DOM 已经建好了。

---

```
下载并执行 app.js
```

**第二步：客户端生成虚拟 DOM**

Vue 执行同一套组件代码，加上 `window.__INITIAL_STATE__` 里的数据，再次运行 render 函数，生成虚拟 DOM：

```
// 客户端内存里的虚拟 DOM
{
  type: 'div',
  props: { class: 'product' },
  children: [
    { type: 'h1', children: ['iPhone 15 Pro'] },
    { type: 'button', children: ['加入购物车'] }
  ]
}
```

**第三步：hydrate，对比虚拟 DOM 和真实 DOM**

`hydrate` 会做一件事：

> 把客户端生成的虚拟 DOM，和浏览器已经存在的真实 DOM 做对比。

如果一致，不会重新创建真实 DOM，而是：

```
button.addEventListener('click', addToCart)
```

直接复用已有真实 DOM，只绑定事件。

如果不一致，会产生 hydration mismatch，框架可能会重新修正真实 DOM。

---

**SSR 里 DOM 生成总结**

```
服务端执行 render    → 生成虚拟 DOM（服务端内存，无真实 DOM）
虚拟 DOM 序列化      → 生成 HTML 字符串（不是真实 DOM）
浏览器解析 HTML      → 浏览器原生生成真实 DOM（不经过框架）
客户端执行 render    → 再次生成虚拟 DOM（客户端内存）
hydrate 对比         → 虚拟 DOM 和真实 DOM 对比
一致则复用           → 不重新生成真实 DOM，只绑定事件
不一致则修正         → 框架修改真实 DOM
```

---

**三、完整对比**

| 阶段 | CSR | SSR |  
|---|  
| 服务端 | 不执行组件，不生成虚拟 DOM | 执行组件，生成虚拟 DOM |  
| 服务端真实 DOM | 无（服务端没有 DOM） |  
| 服务端产出 | 空 HTML 字符串 | 含内容的 HTML 字符串 |  
| 浏览器解析 HTML | 生成空真实 DOM | 生成有内容的真实 DOM |  
| 框架执行 render | 生成虚拟 DOM | 生成虚拟 DOM |  
| 框架创建真实 DOM | 根据虚拟 DOM 创建新的真实 DOM | 不重新创建，hydrate 复用 |  
| 事件绑定 | render 后自动绑定 | hydrate 后绑定 |

---

**四、为什么 SSR 要在客户端再生成一次虚拟 DOM**

因为 Vue / React 内部的响应式系统和事件系统是基于虚拟 DOM 工作的。

客户端如果没有虚拟 DOM：

- 响应式数据变化后不知道该更新哪里
- 事件处理找不到对应节点
- 组件生命周期无法运行

所以客户端必须重新运行 render 生成虚拟 DOM，建立好组件实例、响应式系统、事件处理，才能接管页面后续的交互。

---

**面试版回答**

可以这样说：

> CSR 里，浏览器解析空 HTML 后会先原生生成一个空的真实 DOM。然后 JS 下载完成后，Vue/React 执行组件 render 函数生成虚拟 DOM，再根据虚拟 DOM 调用 `document.createElement` 等浏览器 API 生成真实 DOM，最后插入页面渲染。  
> SSR 里，服务端执行组件 render 生成虚拟 DOM，但服务端没有浏览器 DOM 环境，所以不会生成真实 DOM，而是把虚拟 DOM 序列化成 HTML 字符串返回给浏览器。浏览器解析这段 HTML，原生生成包含业务内容的真实 DOM，用户此时就能看到内容。然后客户端 JS 加载后再次执行 render 生成虚拟 DOM，进行 hydrate，把客户端虚拟 DOM 和已有真实 DOM 对比。如果一致，不重新创建真实 DOM，只复用并绑定事件；如果不一致，框架会修正真实 DOM。

# CSS
CSS 面试最怕“零散背概念”：BFC、盒模型、定位、flex、grid、层叠上下文、回流重绘，每个都听过，但一问就串不起来。

我建议按 **“浏览器如何把 CSS 变成页面”** 这条主线来学。这样知识会成体系。

**CSS 面试学习地图**

1. CSS 基础机制  
    选择器、优先级、继承、层叠、单位、盒模型。
    
2. 布局核心  
    普通流、块级/行内格式化、margin 合并、BFC、浮动、定位、flex、grid。
    
3. 视觉与层叠  
    z-index、层叠上下文、透明度、transform、伪类伪元素。
    
4. 响应式与适配  
    rem、em、vw/vh、媒体查询、移动端适配、1px 问题。
    
5. 渲染与性能  
    回流、重绘、合成、CSS 动画优化、transform/opacity。
    
6. 工程化 CSS  
    scoped、CSS Modules、Sass/Less、Tailwind、命名规范、样式隔离。
## CSS 基础机制

**一、CSS 到底在做什么**

HTML 负责结构：

```
<div class="card">
  <h2>标题</h2>
  <p>内容</p>
</div>
```

CSS 负责描述这些结构应该如何显示：

```
.card {
  width: 300px;
  padding: 16px;
  background: white;
}
```

浏览器会把 HTML 解析成 DOM，把 CSS 解析成 CSSOM，然后结合起来生成渲染树，再布局、绘制、合成。

所以 CSS 的核心问题其实是：

> 哪些元素应用哪些样式？这些样式如何计算？盒子如何布局？最后如何绘制？

---

**二、选择器和优先级**

CSS 选择器用来找到元素。

常见选择器：

```
div {}              /* 标签选择器 */
.card {}            /* 类选择器 */
#app {}             /* id 选择器 */
.card p {}          /* 后代选择器 */
.card > p {}        /* 子代选择器 */
button:hover {}     /* 伪类 */
.box::before {}     /* 伪元素 */
```

面试重点不是背所有选择器，而是要懂 **优先级**。

优先级大致是：

```
!important
内联样式 style
ID 选择器
类选择器 / 属性选择器 / 伪类
标签选择器 / 伪元素
通配符 / 继承
```

比如：

```
<div id="box" class="box" style="color: green;">Hello</div>
```

```
#box {
  color: red;
}

.box {
  color: blue;
}
```

最终是：

```
green
```

因为内联样式优先级更高。

如果没有内联：

```
#box {
  color: red;
}

.box {
  color: blue;
}
```

最终是：

```
red
```

因为 ID 优先级高于 class。

---

**三、优先级怎么算**

可以简单记成四元组：

```
!important > 内联 > ID > 类/伪类/属性 > 标签/伪元素
```

例如：

```
#app .box p {
  color: red;
}
```

包含：

```
1 个 ID
1 个 class
1 个标签
```

```
.card .box p {
  color: blue;
}
```

包含：

```
0 个 ID
2 个 class
1 个标签
```

虽然第二个 class 更多，但第一个有 ID，所以第一个优先级更高。

如果优先级一样，后面的覆盖前面的：

```
.box {
  color: red;
}

.box {
  color: blue;
}
```

最终是：

```
blue
```

这叫层叠。

---

**四、继承**

有些 CSS 属性会从父元素继承到子元素。

比如：

```
body {
  color: #333;
  font-size: 16px;
}
```

子元素里的文字默认也会继承这些样式。

常见可继承属性：

```
color
font-size
font-family
font-weight
line-height
text-align
visibility
```

常见不可继承属性：

```
width
height
margin
padding
border
background
position
display
```

比如：

```
.parent {
  color: red;
  margin: 20px;
}
```

子元素会继承 `color`，但不会继承 `margin`。

面试里可以这样说：

> CSS 中和文本、字体相关的属性通常可以继承，和盒模型、布局相关的属性通常不继承。

---

**五、层叠 Cascade**

CSS 全名是 Cascading Style Sheets，里面的 cascading 就是层叠。

当多个规则作用到同一个元素时，浏览器按这些因素决定最终样式：

```
来源
重要性
优先级
书写顺序
```

比如：

```
p {
  color: red;
}

.text {
  color: blue;
}
```

```
<p class="text">hello</p>
```

最终是蓝色，因为 class 优先级更高。

再比如：

```
.text {
  color: red;
}

.text {
  color: blue;
}
```

最终是蓝色，因为优先级一样时，后写的生效。

---

**六、盒模型**

这是 CSS 面试核心中的核心。

一个元素盒子由四部分组成：

```
content
padding
border
margin
```

从里到外：

```
content box
padding box
border box
margin box
```

CSS：

```
.box {
  width: 200px;
  padding: 20px;
  border: 10px solid #000;
  margin: 30px;
}
```

默认标准盒模型下：

```
content width = 200px
实际占据宽度 = 200 + 20*2 + 10*2 + 30*2
```

注意：

```
width 只表示 content 的宽度
```

---

**七、标准盒模型和 IE 盒模型**

标准盒模型：

```
box-sizing: content-box;
```

此时：

```
元素总宽度 = width + padding + border
```

例如：

```
.box {
  width: 200px;
  padding: 20px;
  border: 10px solid black;
}
```

实际 border box 宽度：

```
200 + 40 + 20 = 260px
```

IE 盒模型，也就是 border-box：

```
box-sizing: border-box;
```

此时：

```
元素总宽度 = width
content 宽度 = width - padding - border
```

同样代码：

```
.box {
  box-sizing: border-box;
  width: 200px;
  padding: 20px;
  border: 10px solid black;
}
```

实际 border box 宽度就是：

```
200px
```

content 宽度是：

```
200 - 40 - 20 = 140px
```

实际项目里通常会全局设置：

```
* {
  box-sizing: border-box;
}
```

这样布局更直观。

---

**八、display 的基础分类**

CSS 布局里要先分清元素盒子类型。

常见：

```
display: block;
display: inline;
display: inline-block;
display: flex;
display: grid;
display: none;
```

**block 块级元素**

特点：

```
独占一行
默认宽度撑满父容器
可以设置 width / height
```

比如：

```
<div></div>
<p></p>
<h1></h1>
```

---

**inline 行内元素**

特点：

```
不会独占一行
宽高由内容决定
设置 width / height 通常无效
垂直方向 margin / padding 对布局影响有限
```

比如：

```
<span></span>
<a></a>
<strong></strong>
```

---

**inline-block**

特点：

```
不独占一行
可以设置 width / height
像行内元素一样排列
```

比如图标、按钮以前常用：

```
.icon {
  display: inline-block;
  width: 20px;
  height: 20px;
}
```

---

**display: none**

元素不显示，也不占据空间。

会影响布局，所以会触发回流。

区别于：

```
visibility: hidden;
```

`visibility: hidden` 是看不见，但仍然占位置，通常只触发重绘。

---

**九、普通流 Normal Flow**

默认情况下，元素按照普通流布局。

块级元素：

```
从上到下排列
每个块独占一行
```

行内元素：

```
从左到右排列
一行放不下会换行
```

比如：

```
<div>块1</div>
<div>块2</div>
<span>行内1</span>
<span>行内2</span>
```

效果大致：

```
块1
块2
行内1 行内2
```

后面讲 BFC、浮动、定位、flex，其实都是在讨论：

> 元素如何脱离或改变普通流的布局规则。

---

**十、这一阶段面试怎么答**

如果面试官问：**你怎么理解 CSS 盒模型？**

可以答：

> CSS 盒模型描述的是元素在页面中占据空间的方式，一个盒子由 content、padding、border、margin 组成。标准盒模型下，`width` 只表示 content 宽度，元素实际宽度还要加上 padding 和 border；而 `box-sizing: border-box` 下，`width` 表示 content + padding + border 的总宽度。实际项目中通常会全局设置 `box-sizing: border-box`，这样布局计算更直观。

如果问：**CSS 优先级怎么计算？**

可以答：

> CSS 优先级主要看选择器权重，`!important` 最高，其次是内联样式、ID 选择器、类选择器/属性选择器/伪类、标签选择器/伪元素。多个规则作用到同一个元素时，先比较优先级，优先级相同则后写的覆盖先写的。另外有些属性会继承，比如字体和文本相关属性；布局和盒模型相关属性通常不会继承。

我们下一步建议进入 CSS 面试最核心的一块：**BFC、margin 合并、浮动清除、块格式化上下文**。这块打通后，很多 CSS 老题会突然变清楚。

## 布局

这部分你要建立一个主线：

> CSS 布局本质是在回答：盒子在页面中如何排列、如何占空间、如何相互影响。

我们按顺序讲：

```
普通流
块级格式化 / 行内格式化
margin 合并
BFC
浮动
定位
flex
grid
```

---

**一、普通流 Normal Flow**

普通流是 CSS 默认布局方式。

如果你什么布局属性都不写，元素就按普通流排布。

普通流里有两类最基础的盒子：

```
块级盒子 block-level box
行内盒子 inline-level box
```

---

**块级元素怎么排**

比如：

```
<div class="box">A</div>
<div class="box">B</div>
<p>C</p>
```

```
.box {
  width: 200px;
  height: 50px;
  background: pink;
}
```

块级元素特点：

```
独占一行
从上到下排列
默认宽度占满包含块
可以设置 width / height
```

所以效果是：

```
A
B
C
```

即使你写了：

```
.box {
  width: 200px;
}
```

它还是会独占一行，只是自身宽度是 200px，剩余空间不会让下一个块级元素上来。

---

**行内元素怎么排**

比如：

```
<span>A</span>
<span>B</span>
<a>C</a>
```

行内元素特点：

```
从左到右排列
一行放不下会换行
宽高由内容决定
设置 width / height 通常无效
```

效果：

```
A B C
```

---

**inline-block**

`inline-block` 介于二者之间：

```
.item {
  display: inline-block;
  width: 100px;
  height: 40px;
}
```

特点：

```
像 inline 一样不独占一行
像 block 一样可以设置宽高
```

---

**二、块级格式化和行内格式化**

你可以理解为两套排版规则。

**块级格式化**

块级盒子在垂直方向一个接一个排列：

```
box1
box2
box3
```

每个块级盒子的左外边距边界，默认和包含块左边界接触。

这就是你之前问的那句话。

---

**行内格式化**

行内盒子在一行里从左到右排列，形成一行一行的 line box。

比如：

```
<p>
  hello <span>world</span> hello world hello world
</p>
```

文字和 `span` 会组成行盒。

如果一行放不下，就换到下一行：

```
hello world hello
world hello world
```

浮动影响的通常就是这些行盒：文字会绕开浮动元素。

---

**三、margin 合并**

margin 合并，也叫 margin collapse。

它只发生在：

```
普通流中的块级元素垂直方向 margin
```

重点是：

```
垂直方向
块级元素
普通流
```

水平 margin 不会合并。

---

**1. 相邻兄弟元素 margin 合并**

```
<div class="a"></div>
<div class="b"></div>
```

```
.a {
  height: 100px;
  margin-bottom: 30px;
  background: red;
}

.b {
  height: 100px;
  margin-top: 50px;
  background: blue;
}
```

你可能以为间距是：

```
30 + 50 = 80px
```

实际是：

```
max(30, 50) = 50px
```

这就是兄弟元素垂直 margin 合并。

---

**2. 父子元素 margin 合并**

```
<div class="parent">
  <div class="child"></div>
</div>
```

```
.parent {
  background: pink;
}

.child {
  margin-top: 50px;
  height: 100px;
  background: red;
}
```

你可能以为子元素距离父元素顶部 50px。

但如果父元素没有：

```
border
padding
inline content
height / min-height
BFC
```

子元素的 `margin-top` 可能会“穿透”父元素，和父元素发生 margin 合并。

表现是：

```
整个 parent 往下移动 50px
而不是 child 在 parent 内部往下移动
```

---

**3. 空块元素自身 margin 合并**

```
.empty {
  margin-top: 20px;
  margin-bottom: 40px;
}
```

如果这个元素没有内容、没有高度、没有 border、没有 padding，它自己的上下 margin 也可能合并。

---

**如何阻止 margin 合并**

常见方式：

```
.parent {
  overflow: hidden;
}
```

或者：

```
.parent {
  display: flow-root;
}
```

或者给父元素加：

```
padding-top: 1px;
```

或者：

```
border-top: 1px solid transparent;
```

更现代、语义更清晰的是：

```
display: flow-root;
```

---

**四、BFC**

BFC 全称：

```
Block Formatting Context
```

块级格式化上下文。

先不要死背定义，先理解它是什么：

> BFC 是页面中的一块独立布局区域。这个区域内部的元素怎么布局，不会影响外部；外部的浮动等也不会影响它内部的布局方式。

或者更口语一点：

> BFC 像一个独立的布局小房间，里面的块级盒子按自己的规则排，外面很多影响进不来。

---

**怎么触发 BFC**

常见方式：

```
overflow: hidden;
overflow: auto;
display: flow-root;
display: inline-block;
display: flex;
display: grid;
position: absolute;
position: fixed;
float: left;
float: right;
```

现在推荐记：

```
display: flow-root;
```

它就是专门用来创建 BFC 的。

---

**BFC 的常见作用**

**1. 清除浮动**

经典问题：

```
<div class="parent">
  <div class="float"></div>
</div>
```

```
.float {
  float: left;
  width: 100px;
  height: 100px;
}
```

如果父元素里只有浮动子元素，父元素高度可能塌陷。

因为浮动元素脱离普通流，不再撑开父元素高度。

解决：

```
.parent {
  display: flow-root;
}
```

或者老写法：

```
.parent {
  overflow: hidden;
}
```

这样父元素形成 BFC，会包含内部浮动，父元素高度被撑开。

---

**2. 阻止 margin 合并**

```
.parent {
  display: flow-root;
}
```

父元素形成 BFC 后，子元素的 margin 不会和父元素外部合并。

---

**3. 避免被浮动元素覆盖**

```
<div class="float"></div>
<div class="content">内容</div>
```

```
.float {
  float: left;
  width: 100px;
  height: 100px;
}

.content {
  display: flow-root;
}
```

`.content` 形成 BFC 后，会避开左侧浮动元素，不会跑到浮动元素下面。

---

**五、浮动 float**

`float` 最早是为了图文环绕。

```
.img {
  float: left;
  width: 120px;
}
```

文字会绕着图片排：

```
[图片] 文字文字文字
[图片] 文字文字文字
      文字文字文字
```

浮动的特点：

```
元素脱离普通流
但仍会影响后面行内内容
后面的文字会环绕它
父元素可能高度塌陷
```

---

**浮动为什么会导致父元素高度塌陷**

```
<div class="parent">
  <div class="child"></div>
</div>
```

```
.child {
  float: left;
  width: 100px;
  height: 100px;
}
```

`.child` 浮动后脱离普通流。

父元素计算高度时，普通流里好像没有子元素。

所以父元素高度变成 0。

解决方式：

```
.parent {
  display: flow-root;
}
```

或者 clearfix：

```
.clearfix::after {
  content: '';
  display: block;
  clear: both;
}
```

---

**六、定位 position**

定位用于改变元素的位置和层级。

常见值：

```
position: static;
position: relative;
position: absolute;
position: fixed;
position: sticky;
```

---

**static**

默认值。

```
position: static;
```

元素在普通流中正常排列。

`top/right/bottom/left` 无效。

---

**relative**

相对定位。

```
.box {
  position: relative;
  top: 20px;
  left: 30px;
}
```

特点：

```
元素仍然占据原来的位置
视觉上相对于自身原位置偏移
不会影响其他元素布局
```

比如一个盒子往下移动 20px，但它原来的坑还留着。

---

**absolute**

绝对定位。

```
.child {
  position: absolute;
  top: 0;
  left: 0;
}
```

特点：

```
脱离普通流
不占据原来的空间
相对于最近的非 static 定位祖先定位
如果没有，就相对于初始包含块
```

常见写法：

```
.parent {
  position: relative;
}

.child {
  position: absolute;
  right: 0;
  top: 0;
}
```

这样 `.child` 相对 `.parent` 定位。

---

**fixed**

固定定位。

```
.box {
  position: fixed;
  right: 20px;
  bottom: 20px;
}
```

特点：

```
脱离普通流
相对于视口定位
页面滚动时位置不变
```

常用于悬浮按钮、固定头部。

---

**sticky**

粘性定位。

```
.header {
  position: sticky;
  top: 0;
}
```

特点：

```
在普通流中占位
滚动到指定阈值后变成类似 fixed
```

常用于吸顶导航。

注意父元素不能有不合适的 `overflow: hidden/auto`，否则 sticky 可能不生效。

---

**七、Flex 布局**

Flex 是一维布局：

```
主轴 + 交叉轴
```

适合处理一行或一列的排列。

```
.container {
  display: flex;
}
```

默认：

```
主轴：水平方向，从左到右
交叉轴：垂直方向
```

---

**容器属性**

```
.container {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
  gap: 12px;
}
```

`flex-direction`：主轴方向

```
row
row-reverse
column
column-reverse
```

`justify-content`：主轴对齐

```
flex-start
center
flex-end
space-between
space-around
space-evenly
```

`align-items`：交叉轴对齐

```
stretch
flex-start
center
flex-end
baseline
```

`flex-wrap`：是否换行

```
nowrap
wrap
```

---

**项目属性**

```
.item {
  flex: 1;
}
```

`flex: 1` 常见含义：

```
flex-grow: 1;
flex-shrink: 1;
flex-basis: 0%;
```

意思是：

```
允许放大
允许缩小
初始基础尺寸为 0
```

常用于等分布局：

```
.item {
  flex: 1;
}
```

三个 item 平分宽度。

---

**经典 flex 居中**

```
.parent {
  display: flex;
  justify-content: center;
  align-items: center;
}
```

水平垂直居中。

---

**八、Grid 布局**

Grid 是二维布局：

```
行 + 列
```

适合整体页面结构、卡片网格。

```
.container {
  display: grid;
  grid-template-columns: 200px 1fr 1fr;
  grid-template-rows: 80px 1fr;
  gap: 16px;
}
```

比如三列布局：

```
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
}
```

自适应卡片：

```
.container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 16px;
}
```

含义：

```
每列最小 240px
空间够就自动增加列数
不够就换行
```

---

**Flex 和 Grid 的区别**

|对比|Flex|Grid|
|---|---|---|
|维度|一维布局|二维布局|
|适合|一行或一列|行列同时控制|
|常用场景|导航栏、按钮组、左右布局|页面整体布局、卡片网格|
|控制方式|主要控制主轴和交叉轴|明确控制行和列|

一句话：

> 一维用 Flex，二维用 Grid。

---

**布局面试版总结**

可以这样答：

> CSS 默认布局是普通流，块级元素从上到下排列并独占一行，行内元素从左到右排列，形成行盒。普通流中的块级元素垂直 margin 可能会发生合并，比如相邻兄弟元素上下 margin 会取较大值，父子元素在没有 border、padding、BFC 等阻隔时也可能合并。
> 
> BFC 是块级格式化上下文，可以理解为独立的布局区域，常见触发方式有 `overflow: hidden`、`display: flow-root`、`display: flex/grid`、`position: absolute/fixed`、`float` 等。BFC 常用来清除浮动、阻止 margin 合并、避免被浮动元素覆盖。
> 
> 浮动最初用于文字环绕，会让元素脱离普通流，但仍影响后面的行内内容，因此可能导致父元素高度塌陷，通常用 `display: flow-root` 或 clearfix 清除浮动。定位里，`relative` 不脱离文档流，偏移后原位置保留；`absolute` 脱离文档流，相对于最近的非 static 定位祖先定位；`fixed` 相对视口固定；`sticky` 是普通流和固定定位的结合。
> 
> Flex 是一维布局，适合一行或一列的对齐和分配空间；Grid 是二维布局，适合同时控制行和列，比如页面结构和卡片网格。面试时可以总结为：一维布局优先 Flex，二维布局优先 Grid。

## 视觉与层叠

这一块面试经常看你有没有真正理解“为什么 z-index 不生效”“为什么元素盖不住别人”“transform 为什么会影响 fixed/absolute”“伪类和伪元素区别”。

主线是：

> 元素不只是二维排版，还有第三个维度：谁盖在谁上面。

---

**一、z-index 是什么**

`z-index` 控制元素在 z 轴上的堆叠顺序。

```
.box1 {
  position: absolute;
  z-index: 1;
}

.box2 {
  position: absolute;
  z-index: 2;
}
```

一般来说，`z-index` 大的元素会盖在 `z-index` 小的元素上。

但注意：

> `z-index` 不是全局无限比较，它只在同一个层叠上下文里比较。

这是最关键的一句。

---

**二、z-index 为什么有时候不生效**

常见原因有两个。

**1. 元素没有定位**

传统情况下，`z-index` 对普通静态元素不生效。

```
.box {
  z-index: 999;
}
```

如果没有写：

```
position: relative;
position: absolute;
position: fixed;
position: sticky;
```

那这个 `z-index` 可能没有效果。

常见写法：

```
.box {
  position: relative;
  z-index: 10;
}
```

不过现代布局里，flex/grid 子项设置 `z-index` 也能生效，即使没有显式定位。但面试基础回答里，一般先说“定位元素 z-index 才生效”即可，再补充 flex/grid 子项例外更准确。

---

**2. 被父级层叠上下文限制**

看这个例子：

```
<div class="parent-a">
  <div class="child-a"></div>
</div>

<div class="parent-b">
  <div class="child-b"></div>
</div>
```

```
.parent-a {
  position: relative;
  z-index: 1;
}

.child-a {
  position: absolute;
  z-index: 999;
}

.parent-b {
  position: relative;
  z-index: 2;
}

.child-b {
  position: absolute;
  z-index: 1;
}
```

你可能以为：

```
child-a z-index 999，会盖住 child-b
```

但实际可能是：

```
parent-b 整体盖住 parent-a
```

因为：

```
parent-a 是一个层叠上下文，z-index: 1
parent-b 是一个层叠上下文，z-index: 2
```

浏览器先比较父级层叠上下文：

```
parent-b > parent-a
```

所以 `child-a` 再大，也只能在 `parent-a` 这个小世界里比较，不能跳出去盖住 `parent-b`。

这就是：

> 子元素的 z-index 再大，也无法突破父级层叠上下文。

---

**三、层叠上下文是什么**

层叠上下文，英文是：

```
Stacking Context
```

你可以理解成：

> 一个独立的 z-index 比较空间。

一个元素形成层叠上下文后，它里面的子元素会先在内部比较层级。整个上下文再作为一个整体，和外面的元素比较。

类似：

```
页面
 ├─ 层叠上下文 A，z-index: 1
 │   ├─ child z-index: 999
 │   └─ child z-index: 1
 │
 └─ 层叠上下文 B，z-index: 2
     ├─ child z-index: 1
```

即使 A 里面有 `999`，A 整体还是低于 B。

---

**四、哪些情况会创建层叠上下文**

常见触发条件：

```
position: relative/absolute/fixed/sticky + z-index 不是 auto
opacity 小于 1
transform 不是 none
filter 不是 none
perspective 不是 none
mix-blend-mode 不是 normal
isolation: isolate
will-change: transform / opacity
contain: paint
display: flex/grid 的子项设置 z-index
```

你面试重点记几个高频：

```
定位元素 + z-index
opacity < 1
transform
filter
position: fixed
position: sticky
will-change
```

---

**五、默认层叠顺序**

在同一个层叠上下文里，大致从下到上：

```
背景和边框
负 z-index 子元素
普通块级元素
浮动元素
行内元素
z-index: auto / 0 的定位元素
正 z-index 的定位元素
```

面试一般不要求背全，但你要知道：

> 后出现的普通元素可能盖住前面的普通元素；定位元素通常比普通流元素更靠上；z-index 为正的定位元素更靠上。

比如：

```
<div class="a"></div>
<div class="b"></div>
```

如果都没有定位，后面的 `.b` 可能在重叠时盖住 `.a`。

---

**六、opacity 透明度**

```
.box {
  opacity: 0.5;
}
```

`opacity` 控制元素整体透明度。

注意是整体，包括：

```
背景
文字
边框
子元素
```

比如：

```
.parent {
  opacity: 0.5;
}
```

那么里面的文字、按钮、图片都会一起半透明。

如果你只想让背景透明，不影响文字，应该用：

```
background: rgba(0, 0, 0, 0.5);
```

而不是：

```
opacity: 0.5;
```

---

**opacity 和层叠上下文**

只要：

```
opacity: 0.99;
```

也会创建新的层叠上下文。

所以这可能影响 `z-index`。

例如某个父元素写了：

```
opacity: 0.99;
```

它的子元素就被限制在这个新的层叠上下文里，可能导致弹窗盖不住外面的元素。

---

**七、transform**

`transform` 用来做变换：

```
.box {
  transform: translateX(100px);
}
```

常见值：

```
translate()
scale()
rotate()
skew()
```

---

**transform 的几个重点**

**1. 不影响普通流布局**

```
.box {
  transform: translateX(100px);
}
```

元素视觉上向右移动，但原本占据的位置还在。

类似 `position: relative` 的视觉偏移：

```
只改变视觉位置，不改变文档流中其他元素的布局
```

---

**2. transform 会创建层叠上下文**

```
.parent {
  transform: translateZ(0);
}
```

它会形成新的 stacking context。

所以子元素的 `z-index` 会被限制在里面。

---

**3. transform 可能影响 fixed 定位**

这是一个很重要的坑。

正常：

```
.fixed {
  position: fixed;
  top: 0;
}
```

是相对于视口定位。

但如果它的祖先元素有：

```
.parent {
  transform: translateX(0);
}
```

那么 `.fixed` 可能不再相对于视口，而是相对于这个 transform 祖先定位。

所以有时候弹窗、吸顶元素放在 transform 容器内部，会出现定位异常。

解决方式通常是：

```
把弹窗挂到 body 下
```

比如 Vue 的 Teleport：

```
<Teleport to="body">
  <Modal />
</Teleport>
```

React 的 Portal：

```
createPortal(<Modal />, document.body)
```

---

**4. transform 适合做动画**

比如：

```
.box {
  transition: transform 0.3s;
}

.box:hover {
  transform: translateY(-10px);
}
```

因为 `transform` 通常不触发布局，很多情况下只走合成，性能比改变 `top/left/width/height` 好。

---

**八、伪类**

伪类用来描述元素的某种状态或结构关系。

写法是单冒号：

```
:hover
:active
:focus
:first-child
:last-child
:nth-child()
:not()
```

例如：

```
button:hover {
  background: blue;
}
```

表示鼠标悬停状态。

```
input:focus {
  border-color: red;
}
```

表示输入框聚焦状态。

```
li:first-child {
  color: red;
}
```

表示第一个子元素。

---

**常见伪类**

```
:hover        鼠标悬停
:active       激活状态
:focus        聚焦
:checked      选中
:disabled     禁用
:first-child  第一个子元素
:last-child   最后一个子元素
:nth-child(n) 第 n 个子元素
:not(selector) 排除某些元素
```

---

**九、伪元素**

伪元素用来创建或选中元素的一部分。

写法通常是双冒号：

```
::before
::after
::first-line
::first-letter
::selection
```

最常见是：

```
.box::before {
  content: '';
  display: block;
}
```

伪元素像是这个元素内部生成了一个虚拟子元素。

比如：

```
.badge::after {
  content: 'NEW';
  color: red;
}
```

```
<span class="badge">消息</span>
```

视觉上类似：

```
<span class="badge">
  消息
  ::after NEW
</span>
```

注意：

> `
> 
> ::before
> 
> `和`::after`必须有`content` 属性，否则通常不会显示。

---

**十、伪类和伪元素区别**

一句话：

```
伪类表示元素的状态或位置
伪元素表示元素的一部分或额外生成的虚拟元素
```

例子：

```
button:hover {}
```

`hover` 是按钮的一种状态，所以是伪类。

```
p::first-line {}
```

`first-line` 是段落的第一行，是元素的一部分，所以是伪元素。

```
.box::before {}
```

创建一个虚拟元素，所以是伪元素。

---

**十一、经典面试题：画三角形**

CSS 三角形常用 border：

```
.triangle {
  width: 0;
  height: 0;
  border-left: 10px solid transparent;
  border-right: 10px solid transparent;
  border-bottom: 20px solid red;
}
```

为什么能形成三角形？

因为元素宽高为 0 时，四个 border 会在中心相交。只保留一个方向有颜色，其他方向透明，就形成三角形。

也可以用伪元素做：

```
.tooltip::after {
  content: '';
  position: absolute;
  left: 20px;
  bottom: -10px;
  width: 0;
  height: 0;
  border-top: 10px solid #333;
  border-left: 10px solid transparent;
  border-right: 10px solid transparent;
}
```

---

**十二、经典问题：为什么弹窗 z-index 很大还是被遮住**

回答路径：

1. 先看弹窗有没有定位：

```
position: fixed;
z-index: 9999;
```

2. 再看父级是否创建了层叠上下文：

```
transform
opacity < 1
filter
position + z-index
```

3. 如果父级层叠上下文层级低，子元素再大也出不去。
    
4. 解决方案：
    

```
提高父级层叠上下文 z-index
或者把弹窗挂到 body 下
```

比如用 Portal / Teleport。

---

**面试版总结**

可以这样说：

> `z-index` 用来控制元素在 z 轴上的堆叠顺序，但它不是全局比较，而是在同一个层叠上下文中比较。常见创建层叠上下文的方式包括定位元素设置非 auto 的 `z-index`、`opacity < 1`、`transform`、`filter`、`position: fixed/sticky` 等。子元素的 z-index 再大，也不能突破父级层叠上下文。
> 
> `opacity` 会让元素整体透明，包括子元素，并且小于 1 时会创建层叠上下文。如果只想背景透明，应该用 `rgba`。`transform` 可以做位移、缩放、旋转，它不会影响普通流布局，但会创建层叠上下文，也可能影响内部 fixed 元素的定位。动画里常用 `transform` 和 `opacity`，因为它们通常性能更好。
> 
> 伪类表示元素的状态或结构位置，比如 `:hover`、`:focus`、`:first-child`；伪元素表示元素的一部分或创建额外的虚拟元素，比如 `::before`、`::after`、`::first-line`。伪元素通常需要 `content` 才能显示。
> 

好，我们进入 CSS 第四块：**响应式与适配**。

这一块面试要抓住一个主线：

> 不同设备屏幕大小、像素密度、使用场景都不同，CSS 需要让页面在不同环境下保持可读、可用、布局合理。

我们按这几个点讲：

```
px
em
rem
vw / vh
媒体查询
移动端适配
1px 问题
```

---

**一、先理解 px**

`px` 是 CSS 像素，不一定等于设备物理像素。

比如 iPhone 的 DPR 是 2 或 3。

```
1 CSS px 可能对应 2x2 或 3x3 个物理像素
```

所以：

```
.box {
  width: 100px;
}
```

这里的 `100px` 是 CSS 像素，不是屏幕硬件的 100 个发光点。

---

**二、DPR 是什么**

DPR 全称：

```
Device Pixel Ratio
```

设备像素比。

公式：

```
DPR = 物理像素 / CSS 像素
```

比如某手机：

```
CSS 视口宽度：375px
物理像素宽度：750px
```

那么：

```
DPR = 750 / 375 = 2
```

这就是为什么设计稿经常是 750px，而 CSS 写 375px。

---

**三、viewport 是什么**

移动端必须写：

```
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

含义：

```
layout viewport 宽度等于设备宽度
初始缩放比例为 1
```

如果不写，移动端浏览器可能会用一个默认的宽视口，比如 980px，然后把页面缩小显示，导致页面看起来很小。

所以移动端适配第一步就是 viewport。

---

**四、em**

`em` 是相对于当前元素或父元素字体大小的单位。

看例子：

```
.parent {
  font-size: 20px;
}

.child {
  width: 10em;
}
```

如果 `.child` 自己没有设置 `font-size`，继承父元素的 `20px`。

那么：

```
10em = 10 * 20px = 200px
```

如果 `.child` 自己设置：

```
.child {
  font-size: 16px;
  width: 10em;
}
```

那么：

```
10em = 10 * 16px = 160px
```

所以 `em` 的问题是：

> 它会受到当前元素 font-size 影响，嵌套多了容易计算混乱。

---

**em 常用场景**

`em` 适合做和文字大小相关的局部尺寸。

比如按钮：

```
.button {
  font-size: 16px;
  padding: 0.5em 1em;
}
```

如果按钮字体变大，padding 也跟着变大，比例保持自然。

---

**五、rem**

`rem` 全称：

```
root em
```

它相对于根元素 `html` 的字体大小。

```
html {
  font-size: 16px;
}

.box {
  width: 10rem;
}
```

那么：

```
10rem = 160px
```

不管嵌套多深，`rem` 都只看：

```
html {
  font-size: ?
}
```

所以 `rem` 比 `em` 更适合做全局适配。

---

**rem 移动端适配思路**

如果设计稿是 750px，常见方案是：

```
function setRem() {
  const width = document.documentElement.clientWidth
  document.documentElement.style.fontSize = width / 10 + 'px'
}

setRem()
window.addEventListener('resize', setRem)
```

如果屏幕宽度是 375px：

```
html font-size = 375 / 10 = 37.5px
```

那么设计稿里的 75px，可以写成：

```
75 / 75 = 1rem
```

不同团队换算规则不同，但核心是：

> 根据屏幕宽度动态设置根字号，然后页面尺寸用 rem 表示，实现等比例缩放。

---

**rem 的优点和缺点**

优点：

```
全局统一缩放
适合移动端等比例适配
比 em 稳定
```

缺点：

```
需要 JS 或构建工具换算
大屏下可能过度放大
不是所有场景都适合等比例缩放
```

所以实际项目里经常会限制最大宽度：

```
const width = Math.min(document.documentElement.clientWidth, 750)
```

避免平板或 PC 上字体巨大。

---

**六、vw / vh**

`vw` 和 `vh` 是视口单位。

```
1vw = 视口宽度的 1%
1vh = 视口高度的 1%
```

比如视口宽度是 375px：

```
1vw = 3.75px
100vw = 375px
```

示例：

```
.banner {
  width: 100vw;
  height: 50vw;
}
```

表示 banner 宽度等于视口宽度，高度是视口宽度的一半。

---

**vw 适配设计稿**

如果设计稿宽度是 750px。

设计稿里某个元素宽 75px。

换成 vw：

```
75 / 750 * 100 = 10vw
```

所以：

```
.box {
  width: 10vw;
}
```

屏幕越宽，元素越宽。

---

**vh 的坑**

移动端 `100vh` 有时候不好用。

因为浏览器地址栏会收起 / 展开，导致可视区域变化。

比如：

```
.page {
  height: 100vh;
}
```

在移动端可能出现底部被遮挡或跳动。

现在可以了解新单位：

```
100dvh
100svh
100lvh
```

简单理解：

```
dvh：动态视口高度
svh：小视口高度
lvh：大视口高度
```

面试中提一句即可，不需要深挖。

---

**七、媒体查询**

媒体查询用于根据屏幕宽度、设备特性应用不同样式。

```
.container {
  width: 1200px;
}

@media (max-width: 768px) {
  .container {
    width: 100%;
    padding: 16px;
  }
}
```

意思是：

> 当视口宽度小于等于 768px 时，应用里面的样式。

---

**常见断点**

```
/* 手机 */
@media (max-width: 767px) {}

/* 平板 */
@media (min-width: 768px) and (max-width: 1023px) {}

/* 桌面 */
@media (min-width: 1024px) {}
```

响应式布局常见策略：

```
小屏一列
中屏两列
大屏三列或四列
```

示例：

```
.grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
}

@media (max-width: 1024px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 600px) {
  .grid {
    grid-template-columns: 1fr;
  }
}
```

---

**八、移动端适配常见方案**

**1. viewport + 百分比 / flex**

```
.layout {
  display: flex;
}

.left {
  flex: 1;
}

.right {
  flex: 2;
}
```

适合弹性布局。

---

**2. rem 方案**

根据屏幕宽度动态设置 `html font-size`。

适合按设计稿等比例还原。

---

**3. vw 方案**

直接用视口宽度计算。

```
.box {
  width: 20vw;
}
```

或者配合 PostCSS 插件自动把 px 转 vw。

---

**4. 媒体查询**

适合多端响应式，比如 PC + 平板 + 手机。

---

**5. max-width 居中容器**

很多移动端页面会这样：

```
.app {
  max-width: 750px;
  margin: 0 auto;
}
```

避免在大屏上无限拉伸。

---

**九、1px 问题**

移动端常说的 1px 问题是：

> 在高 DPR 屏幕上，CSS 的 1px 会对应多个物理像素，看起来比设计稿里的 1 物理像素更粗。

例如 DPR = 2：

```
1 CSS px = 2 物理像素
```

所以：

```
border: 1px solid #ddd;
```

在视觉上可能显得偏粗。

---

**解决方案 1：transform scale**

用伪元素画边框，然后缩放。

```
.hairline {
  position: relative;
}

.hairline::after {
  content: '';
  position: absolute;
  left: 0;
  bottom: 0;
  width: 100%;
  border-bottom: 1px solid #ddd;
  transform: scaleY(0.5);
  transform-origin: 0 100%;
}
```

在 DPR = 2 的屏幕上，把 1px 缩成 0.5 CSS px，视觉上接近 1 物理像素。

如果 DPR = 3，可以用：

```
transform: scaleY(0.333);
```

但通常项目里会用统一 mixin 或组件处理。

---

**解决方案 2：使用 viewport 缩放方案**

有些老移动端方案会根据 DPR 设置 viewport scale。

比如 DPR = 2 时：

```
<meta name="viewport" content="initial-scale=0.5">
```

这样 1 CSS px 会更接近 1 物理像素。

但这种方案现在不如以前常用，维护成本高。

---

**解决方案 3：直接接受**

现在很多业务里不一定强求物理 1px。

因为不同设备、不同屏幕显示效果差异很大。

如果对视觉要求不是特别高，直接：

```
border: 1px solid #eee;
```

也可以接受。

---

**十、单位怎么选**

|单位|适合场景|
|---|---|
|`px`|固定尺寸、边框、图标|
|`em`|和当前字体大小关联的局部尺寸，比如按钮 padding|
|`rem`|全局尺寸、移动端等比例适配|
|`%`|相对于父元素的弹性尺寸|
|`vw/vh`|相对于视口的尺寸，适合全屏、响应式|
|`fr`|Grid 中分配剩余空间|

---

**面试版总结**

可以这样说：

> 响应式和移动端适配的核心是让页面在不同屏幕尺寸和不同 DPR 设备上保持合理显示。`px` 是 CSS 像素，不一定等于物理像素；DPR 表示物理像素和 CSS 像素的比例。移动端通常需要设置 viewport，让布局视口等于设备宽度。
> 
> `em` 相对于当前元素字体大小，适合做局部跟随文字缩放的尺寸；`rem` 相对于根元素字体大小，适合做全局适配；`vw/vh` 相对于视口宽高，适合直接按屏幕比例布局。响应式布局常用媒体查询，根据不同屏幕宽度切换布局，比如手机一列、平板两列、桌面多列。
> 
> 移动端适配常见方案有 viewport + flex/百分比、rem 等比例缩放、vw 适配、媒体查询等。1px 问题是因为高 DPR 屏幕下 1 CSS px 对应多个物理像素，边框看起来偏粗，常用伪元素加 `transform: scaleY(0.5)` 的方式模拟更细的边框。