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