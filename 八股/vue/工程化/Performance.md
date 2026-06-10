**Performance 面板里最核心的一部分就是火焰图**，但 Performance 不只等于火焰图。

更准确地说：

```
Chrome DevTools Performance = 运行时性能分析工具
火焰图 = Performance 里用来展示主线程任务调用和耗时的一种视图
```

它主要用来分析：

```
页面为什么卡
JS 哪里执行久
渲染哪里慢
动画为什么掉帧
点击后为什么没反应
首屏加载期间发生了什么
```

**1. Performance 看什么**

Chrome DevTools 里打开：

```
F12 → Performance → Record
```

录制一段操作后，你会看到大概这些区域：

```
FPS：帧率
CPU：CPU 使用情况
Network：资源加载
Screenshots：页面截图
Main：主线程任务
Timings：关键时间点
Experience：布局偏移等体验问题
Summary：当前选中任务的统计
```

其中你常说的“火焰图”，主要在：

```
Main 主线程区域
```

---

**2. 火焰图是什么**

火焰图用一条条横向色块表示任务。

大致长这样：

```
Task
  ├─ Evaluate Script
  │   ├─ functionA
  │   │   └─ functionB
  │   └─ functionC
  ├─ Recalculate Style
  ├─ Layout
  └─ Paint
```

在图上表现为一层一层的色块。

你可以这样理解：

```
横向宽度 = 耗时长短
纵向层级 = 调用关系
颜色 = 任务类型
```

比如一个很宽的 JS 色块，就说明这段 JS 执行很久。

---

**3. 火焰图怎么看**

重点看三个东西。

第一，看有没有 Long Task。

Chrome 通常会标出超过 `50ms` 的长任务。

```
一帧理想时间：约 16.7ms
长任务阈值：50ms
```

如果一个任务跑了 200ms，用户在这期间点击、滚动、输入，都可能没反应。

比如：

```
button.addEventListener('click', () => {
  heavyCalculate()
  updateDom()
})
```

如果 `heavyCalculate()` 很慢，Performance 里就能看到点击后主线程被 JS 占满。

---

第二，看耗时是在 JS、Layout 还是 Paint。

常见颜色大致表示：

```
黄色：JavaScript 执行
紫色：样式计算 / 布局 Layout
绿色：绘制 Paint / 合成
蓝色：HTML 解析 / 加载相关
```

如果黄色很多：

```
JS 太重
函数执行太久
依赖库太大
循环或递归太多
```

如果紫色很多：

```
频繁改 DOM
频繁触发布局
读写布局交叉
元素数量太多
```

如果绿色很多：

```
绘制成本高
阴影、滤镜、渐变、大面积重绘
图片或复杂视觉效果太重
```

---

第三，看调用栈。

比如你点开一个很宽的任务，可能看到：

```
Task
  Event: click
    handleClick
      renderList
        Array.map
          createElement
      Recalculate Style
      Layout
```

这表示：

```
点击事件触发 handleClick
handleClick 里执行 renderList
renderList 创建大量 DOM
然后浏览器重新计算样式和布局
```

这时优化方向就比较明确：

```
减少一次性渲染数量
使用虚拟列表
拆分任务
避免频繁 DOM 操作
```

---

**4. Performance 和 Lighthouse 区别**

很多人会混。

```
Performance 面板：
手动录制，分析具体运行过程。
适合定位“哪里慢”。

Lighthouse：
自动打分，给性能指标和建议。
适合整体评估“慢不慢”。
```

比如：

```
Lighthouse 告诉你 LCP 慢
Performance 帮你找 LCP 为什么慢
```

---

**5. 一个典型卡顿例子**

代码：

```
button.addEventListener('click', () => {
  const list = document.querySelector('.list')

  for (let i = 0; i < 10000; i++) {
    const item = document.createElement('div')
    item.textContent = `Item ${i}`
    list.appendChild(item)
  }
})
```

点击后页面卡顿。

Performance 里可能看到：

```
Event: click
  Function Call
  Recalculate Style
  Layout
  Paint
```

耗时很长。

原因：

```
一次性创建太多 DOM
浏览器需要重新计算样式
重新布局
重新绘制
主线程被占用
```

优化：

```
button.addEventListener('click', () => {
  const list = document.querySelector('.list')
  const fragment = document.createDocumentFragment()

  for (let i = 0; i < 10000; i++) {
    const item = document.createElement('div')
    item.textContent = `Item ${i}`
    fragment.appendChild(item)
  }

  list.appendChild(fragment)
})
```

更进一步：

```
使用虚拟列表
分页加载
分片渲染
```

---

**6. 什么是强制同步布局**

这是 Performance 里很常见的问题。

坏例子：

```
const box = document.querySelector('.box')

box.style.width = '200px'
console.log(box.offsetWidth)

box.style.height = '300px'
console.log(box.offsetHeight)
```

你先写样式，又马上读布局属性：

```
offsetWidth
offsetHeight
getBoundingClientRect()
clientWidth
scrollTop
```

浏览器为了给你准确结果，必须立刻计算布局。这叫：

```
Forced reflow / forced layout
```

Performance 里会看到 Layout 频繁出现。

更好：

```
const box = document.querySelector('.box')

box.style.width = '200px'
box.style.height = '300px'

const rect = box.getBoundingClientRect()
console.log(rect)
```

原则：

```
批量读
批量写
不要读写交叉
```

---

**7. 动画卡顿怎么看**

如果动画卡顿，Performance 里重点看：

```
FPS 是否掉得厉害
Main 线程有没有长任务
每一帧是否频繁 Layout/Paint
动画属性是不是 left/top/width/height
```

坏写法：

```
box.style.left = `${x}px`
```

可能引发 Layout。

更好：

```
box.style.transform = `translateX(${x}px)`
```

通常走合成，成本低。

如果你看到每帧都有：

```
Recalculate Style
Layout
Paint
```

说明动画每帧都在重新布局/绘制。

理想情况是更多走：

```
Composite
```

---

**8. 首屏加载怎么看**

Performance 也可以分析首屏。

重点看这些标记：

```
FP：First Paint
FCP：First Contentful Paint
LCP：Largest Contentful Paint
DOMContentLoaded
Load
```

如果 FCP 很晚，可能是：

```
HTML 返回慢
CSS 阻塞
JS 阻塞渲染
资源太大
```

如果 LCP 很晚，可能是：

```
首屏大图加载慢
接口数据回来太晚
主线程被 JS 阻塞
字体阻塞
服务端响应慢
```

---

**9. Bottom-Up、Call Tree、Event Log 是什么**

选中一段时间后，下面通常有几个视图。

`Summary`：

```
告诉你这段时间 JS、渲染、绘制分别花了多少时间
```

`Bottom-Up`：

```
按耗时从大到小统计函数
适合找最耗时的函数
```

比如：

```
renderList  300ms
formatData  120ms
calculate   80ms
```

`Call Tree`：

```
按调用关系展示
适合看是谁调用了谁
```

`Event Log`：

```
按时间顺序列出事件
适合看完整执行流程
```

排查时通常这样：

```
先看 Main 火焰图找长任务
再看 Bottom-Up 找最耗时函数
再看 Call Tree 看调用来源
```

---

**10. 实战排查顺序**

遇到页面卡顿，可以按这个顺序：

```
1. 打开 Performance
2. 点击 Record
3. 操作页面，比如点击按钮、滚动、打开弹窗
4. Stop
5. 找红色/很宽的长任务
6. 看 Main 线程火焰图
7. 判断是 JS、Layout、Paint 哪类耗时
8. 用 Bottom-Up 找具体函数
9. 回代码里优化
10. 再录一次对比
```

---

**11. 常见结论和对应优化**

```
JS 长任务多：
拆分任务、减少计算、Web Worker、减少依赖体积

Layout 很多：
减少 DOM 数量、避免读写交叉、用 transform 替代 left/top

Paint 很多：
减少阴影/filter/大面积渐变、减少重绘区域

动画掉帧：
使用 transform/opacity、requestAnimationFrame、避免主线程长任务

首屏慢：
路由懒加载、压缩资源、优化图片、减少阻塞 JS/CSS

列表滚动卡：
虚拟列表、分页、减少 DOM 节点
```

一句话总结：

```
Performance 是运行时录像机，火焰图是录像里的主线程执行地图。
```

你看火焰图，主要就是找：

```
哪个任务很长
它为什么长
是 JS 慢、布局慢，还是绘制慢
最后对应回代码优化
```