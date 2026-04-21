在前端开发中，**BOM** 和 **DOM** 是两个核心概念，它们共同构成了 JavaScript 与浏览器及网页内容交互的基础。简单来说，JavaScript 通过操作 BOM 和 DOM 来实现网页的动态效果和用户交互。

### 📄 DOM (文档对象模型)

DOM 的全称是 **Document Object Model**，即文档对象模型。你可以把它想象成浏览器将你的 HTML 或 XML 文档解析后，在内存中创建的一棵“对象树”。

- **核心作用**：DOM 提供了一套标准的编程接口（API），让 JavaScript 能够访问、修改网页的内容、结构和样式。没有 DOM，JavaScript 就无法“看见”和“操作”网页上的元素。
- **树状结构**：在 DOM 中，HTML 文档的每个部分都被抽象成一个“节点”（Node）。
    - 整个文档是一个 `document` 节点。
    - 每个 HTML 标签（如 `<div>`, `<p>`）是一个“元素节点”。
    - 标签内的文本是“文本节点”。
    - 标签的属性（如 `id`, `class`）是“属性节点”。

这些节点通过父子、兄弟等关系组织成一棵树，即 **DOM 树**。

**举个例子**：  
对于以下 HTML 代码：

```html
<html>
  <head>
    <title>我的页面</title>
  </head>
  <body>
    <h1>欢迎</h1>
    <p>这是一个段落。</p>
  </body>
</html>
```

浏览器会构建出如下的 DOM 树结构：

```文本
document
 └── html
      ├── head
      │    └── title
      │         └── "我的页面" (文本节点)
      └── body
           ├── h1
           │    └── "欢迎" (文本节点)
           └── p
                └── "这是一个段落。" (文本节点)
```

通过 JavaScript，我们可以操作这棵树，例如：

```javascript
// 获取 h1 元素并修改其文本内容
const titleElement = document.querySelector('h1');
titleElement.textContent = '欢迎学习 DOM！';
```

### 🌐 BOM (浏览器对象模型)

BOM 的全称是 **Browser Object Model**，即浏览器对象模型。它提供了一套与浏览器窗口进行交互的对象和方法。

- **核心作用**：BOM 让 JavaScript 能够控制和操作浏览器本身，而不是网页内容。它关注的是浏览器窗口的各种行为。
- **核心对象**：BOM 的核心是 `window` 对象，它代表浏览器窗口本身，同时也是 JavaScript 中的全局对象。所有全局变量和函数实际上都是 `window` 对象的属性和方法。

BOM 包含许多有用的子对象：

| 对象              | 描述              | 常用功能示例                                 |
| --------------- | --------------- | -------------------------------------- |
| **`window`**    | 浏览器窗口的顶层对象。     | `alert()`, `confirm()`, `setTimeout()` |
| **`location`**  | 提供当前窗口的 URL 信息。 | `window.location.href = '...'` (页面跳转)  |
| **`navigator`** | 包含浏览器的信息。       | `window.navigator.userAgent` (获取用户代理)  |
| **`history`**   | 提供浏览器历史记录的操作。   | `window.history.back()` (后退一页)         |
| **`screen`**    | 提供用户屏幕的信息。      | `window.screen.width` (获取屏幕宽度)         |

### 🤝 DOM 与 BOM 的关系

可以把它们的关系理解为：

- **BOM** 是浏览器提供的一整套能力，用于操作浏览器窗口。
- **DOM** 是 BOM 的一部分，是 `window` 对象下的一个属性（`window.document`），专门负责管理网页内容。

一个生动的比喻是：

- **BOM** 就像是整个**房子**，它包含了墙壁、门窗、水电等所有基础设施。
- **DOM** 就像是房子里的**家具和摆设**，你可以随意移动、更换它们来改变房间的布局和样貌。
- **JavaScript** 就是你，你既可以通过 BOM 来开关窗户（操作浏览器），也可以通过 DOM 来 rearrange 家具（操作网页内容）。