## 1. Nuxt 与项目架构

1. 为什么这个项目选择 Nuxt，而不是普通 Vue3 + Vite？
2. Nuxt 的 SSR、CSR、SSG 你了解吗？
3. 你的项目中是否真的用到了 SSR？
4. Nuxt 的目录结构和普通 Vue 项目有什么区别？
5. 服务端接口是 Nuxt server routes 实现的吗？
6. API Key 放在哪里？如何避免暴露到前端？
7. 如果部署到生产环境，前后端如何分层？
8. Nuxt 中如何处理环境变量？
9. 页面刷新后，历史会话如何恢复？
10. 如果让你把这个项目部署上线，你会怎么部署？

## 2. 文档上传与解析

### 一、整体流程

面试官问“文档上传和解析怎么做的”，你不要直接分散回答，可以先给一个完整链路：

```
用户选择文件  ↓前端校验文件类型 / 大小 / 数量  ↓上传文件或读取文件内容  ↓根据文件类型选择不同解析器  ↓解析出纯文本或结构化文本  ↓文本清洗  ↓按规则分块 chunk  ↓调用 embedding 模型向量化  ↓存储文档、chunk、向量、metadata  ↓更新文档状态为完成
```

面试可以这样说：

> 我这块不是简单上传文件，而是把文档处理拆成几个阶段：文件校验、上传、解析、分块、向量化、入库和状态更新。每个阶段都有独立状态，如果某一步失败，会记录失败原因并允许用户重试。删除文档时，也会同步删除该文档关联的 chunk 和向量数据，避免检索时召回已经删除的内容。

### 二、PDF、DOCX、TXT、Markdown解析

#### 1. TXT

TXT 最简单，直接读取文本内容。

前端可以用：

```js
const text = await file.text()
```

需要注意编码问题：

```plaintxt
UTF-8 正常GBK / ANSI 可能乱码超大 TXT 不能一次性直接渲染
```

面试回答：

> TXT 文件一般可以直接通过 File API 的 `file.text()` 读取内容，然后做换行、空格、特殊字符清洗。需要注意编码格式，如果不是 UTF-8，可能出现乱码，生产环境可以放到服务端用更稳定的编码识别方案处理。

#### 2. Markdown

Markdown 本质上也是文本文件，可以先读原始内容。

```js
const markdown = await file.text()
```

但 Markdown 不建议直接丢掉结构，因为它的标题、列表、代码块对 RAG 很有价值。

Markdown 标题可以作为 **chunk metadata**：

```markdown
# Vue
## 响应式原理
### ref 和 reactive
```

对应 chunk 可以记录：

```ts
{  content: 'ref 是...',  
   metadata: {    
	   h1: 'Vue',    
	   h2: '响应式原理',    
	   h3: 'ref 和 reactive'  
   }
}
```

面试回答：

> Markdown 我会保留标题层级。因为标题层级能够提供上下文，比如某个 chunk 本身只有一段解释，但它所属的一级标题和二级标题能告诉模型这段内容属于哪个主题。分块时，我会尽量按标题、段落进行切分，并把标题路径作为 metadata 存起来。后续检索召回时，可以把标题一起拼进 prompt，提高回答的上下文完整性。

比如 prompt 里可以拼：

```
来源文档：Vue学习笔记.md
章节：Vue > 响应式原理 > ref 和 reactive
内容：xxx
```

这样比只给模型一段孤立文本效果更好。

#### 3. PDF

PDF 解析通常用 PDF 解析库，比如：

```txt
pdf.js
pdf-parse
服务端解析库
```

流程大概是：

```txt
读取 PDF 文件
	↓
按页解析文本
	↓
合并每页文本
	↓
保留页码信息
	↓
清洗页眉页脚、重复空格、断行
```

**PDF 不是天然结构化文本**
##### 1. 换行错乱

PDF 里一行文字可能被拆成多个 text item，解析出来会变成：

```
这是一个很长
的句子
被错误换行
```

需要做行合并。
##### 2. 顺序错乱

双栏 PDF、论文 PDF、带侧边栏的 PDF，解析顺序可能变成：

```
左栏第一行
右栏第一行
左栏第二行
右栏第二行
```

导致语义混乱。
##### 3. 页眉页脚重复

每页都有标题、页码、版权信息，会干扰 embedding。

例如：

```
公司内部文档
第 1 页
公司内部文档
第 2 页
```

需要清洗重复模式。
##### 4. 表格结构丢失

PDF 里的表格解析后可能变成普通文本，列关系丢失。
##### 5. 扫描版 PDF 没有文本层

有些 PDF 实际上是一张张图片，普通文本解析拿不到内容。

处理方式是 OCR：

```
PDF 转图片
	↓
每页图片送 OCR
	↓
识别文字
	↓
按页合并
	↓
再进入分块和向量化流程
```

可以说：

> 扫描版 PDF 本质上是图片，需要先把每一页转成图片，再通过 OCR 识别文字。前端可以做初步判断，比如普通解析后文本长度很短，或者大部分页面没有 text layer，就提示用户这是扫描版文档。真正生产环境里，我更倾向于把 OCR 放在服务端或异步任务里处理，因为 OCR 耗时长、资源消耗大，不适合完全放在前端主线程。
##### 6. 特殊字符乱码

中文、数学公式、特殊符号可能解析异常。

面试回答：

> PDF 我一般会按页解析文本，解析时保留页码信息。因为后续 RAG 检索时，如果某个 chunk 来自第几页，可以在回答里展示来源。PDF 解析之后还要做清洗，比如去掉页眉页脚、处理异常换行、合并被 PDF 排版拆开的句子。

#### 4. DOCX

DOCX 不是纯文本，它本质上是压缩包，里面包含 XML。

常见方式：

```txt
mammoth.js
服务端 docx 解析库
```

解析内容包括：

```txt
段落
标题
列表
表格
图片
说明
```

面试回答：

> DOCX 我会使用专门的解析库，比如 mammoth 这类工具，把 docx 转成文本或者 Markdown-like 结构。解析时重点保留段落和标题层级。如果有表格，要么转成 Markdown 表格，要么转成结构化文本，否则直接拼接容易丢失语义。

### 三、删除文档

这是 RAG 项目很关键的问题。

不能只删前端列表，也不能只删原文件。

一个文档通常对应：

```
document 表
chunk 表
vector 表
file 存储
conversation 引用记录，视情况处理
```

删除流程：

```
用户点击删除
  ↓
前端二次确认
  ↓
调用删除文档 API
  ↓
后端根据 documentId 查所有 chunkId
  ↓
删除向量库中对应向量
  ↓
删除 chunk 记录
  ↓
删除 document 记录
  ↓
删除原始文件
  ↓
前端更新列表
```

面试回答：

> 删除文档时，我会以 documentId 作为主键，不只删除文档记录，还要同步删除它关联的 chunks 和向量数据。每个 chunk 在入库时都会带上 documentId，所以删除时可以按 documentId 过滤删除向量库里的相关向量。这样可以避免用户删除了文档，但后续问答时仍然召回旧内容。

还可以补充一致性问题：

> 如果删除向量失败，我不会直接把前端状态改成删除成功，而是保留失败提示或者做后台补偿任务。因为文档记录和向量数据必须保持一致，否则会出现脏召回。

### 四、文件名重复

不要只用文件名作为唯一标识。

推荐：

```
documentId 唯一
fileHash 判断内容
filename 只是展示字段
```

情况分几种：
#### 1. 文件名相同，内容也相同

可以提示：

```
该文件已存在，是否跳过 / 覆盖 / 重新上传？
```

也可以做秒传。

#### 2. 文件名相同，内容不同

不能直接覆盖，应该生成新的 documentId。

展示时可以改名：

```
简历.pdf简历(1).pdf
```

#### 3. 内容相同，文件名不同

可以根据 hash 判断重复，避免重复入库。

面试回答：

> 文件名不能作为唯一标识，我会用 documentId 做主键，用 fileHash 判断内容是否重复。文件名只是展示字段。如果文件名相同但内容不同，会创建新的文档记录，比如展示为 xxx(1).pdf。如果 hash 相同，则可以提示文件已存在，或者直接走秒传逻辑，避免重复解析和向量化。

### 五、文档过大

可以从三层说：
#### 1. 上传前限制

```js
const MAX_SIZE = 50 * 1024 * 1024
if (file.size > MAX_SIZE) {
  message.error('文件过大')
}
```

限制：

```
单文件大小
总文件大小
一次上传数量
文件类型
```
#### 2. 处理阶段优化

大文档不要主线程处理：

```
Web Worker 分块
分片上传
进度展示
异步任务
```
#### 3. 用户体验

```
提示预计耗时
展示阶段
进度允许取消
失败可重试
```

面试回答：

> 前端会在上传前限制文件类型、单文件大小、总大小和数量。比如超过限制直接拦截，并提示用户拆分文档。对于比较大的文件，我不会在主线程里一次性解析和分块，而是结合分片上传、Web Worker 和阶段性进度展示，避免页面卡死。如果生产环境文档特别大，我会把解析和向量化放到服务端异步任务里处理。

### 六、用户上传多个文件

这个问题要答得有取舍。

> 上传可以并行，但解析和向量化要限制并发。

原因：

```
全部串行：稳定，但慢
全部并行：快，但容易占满网络、CPU、接口限流
限制并发：体验和稳定性更平衡
```

例如：

```
最多同时上传 3 个文件
每个文件内部分片并发 3-5 个
解析任务 1-2 个
并发embedding 请求按接口限流控制
```

面试回答：

> 多文件上传我不会完全串行，也不会无限并行，而是做并发控制。文件上传阶段可以允许 2 到 3 个文件并行，提高效率；但是解析、分块和向量化会限制并发，尤其是 embedding 接口可能有 QPS 限制，如果全部并发容易失败。每个文件都有独立状态，某个文件失败不会影响其他文件继续处理。

可以再加一句：

> 如果用户上传 10 个文件，我会用任务队列维护状态，每个任务独立流转：pending、uploading、parsing、chunking、embedding、completed、failed。

### 七、metadata 设计

每个 chunk 不只是 content，还应该带 metadata：

```js
{
  chunkId: 'chunk_001',  
  documentId: 'doc_001',  
  fileName: 'Vue学习笔记.md',  
  pageNumber: 3,  
  headingPath: ['Vue', '响应式原理'],  
  chunkIndex: 0,  
  content: '...'
}
```

这样后续可以展示来源

## 3. 文本分块策略

1. 为什么要做文本分块？
2. 你说用了“按段落、按句子、按固定长度”的递归分块策略，具体流程是什么？
3. 分块大小怎么确定？
4. overlap 重叠长度有什么作用？
5. overlap 太大或太小分别有什么问题？
6. 中文文本按句子切分时如何识别句号、问号、分号？
7. 代码块、表格、列表是否会被错误切开？
8. 如何保留 chunk 和原文档之间的映射关系？
9. 如何给 chunk 添加 metadata？
10. 如果一个段落超过最大 chunk size，怎么处理？
11. 分块策略会如何影响召回质量？
12. 你有没有做过不同 chunk size 的效果对比？
13. RAG 中为什么不能直接把整篇文档塞给大模型？
14. 如果文档有目录结构，是否应该按标题分块？
15. 你如何判断当前分块策略是好是坏？

## 4. Web Worker

1. 为什么要用 Web Worker？
2. Web Worker 和主线程之间如何通信？
3. Worker 能不能直接操作 DOM？
4. 大文本分块为什么会阻塞主线程？
5. 你是怎么把文件内容传给 Worker 的？
6. postMessage 传大对象会不会有性能问题？
7. structured clone 是什么？
8. 是否用到了 Transferable Objects？
9. Worker 执行出错如何捕获？
10. 用户取消上传时，Worker 如何终止？
11. 多个文件同时分块时，是一个 Worker 还是多个 Worker？
12. Worker 文件在 Vite/Nuxt 中怎么引入？
13. Web Worker 会增加打包复杂度吗？
14. 移动端浏览器对 Worker 支持如何？
15. 如果不用 Worker，还有什么优化方案？

## 5. 向量化与语义检索

1. 什么是 embedding？
2. 文档如何从文本变成向量？
3. 你用的是 DashScope 的哪个模型？
4. 向量存储在哪里？前端、本地数据库还是服务端？
5. 语义检索的流程是什么？
6. 用户提问后，是先向量化问题，还是直接关键词匹配？
7. topK 检索条数怎么设置？
8. 检索结果如何排序？
9. 相似度阈值怎么设定？
10. 如果召回内容不相关，怎么优化？
11. 是否支持混合检索，比如关键词 + 向量？
12. RAG 的回答如何引用原文来源？
13. 如果多个 chunk 内容相似，如何去重？
14. 文档更新后，旧向量如何处理？
15. embedding 模型和生成模型有什么区别？

## 6. SSE 流式对话

### 为什么使用 SSE

可以这样答：

> 因为 RAG 问答场景本质上是“客户端发起一次提问，服务端持续向客户端推送模型生成结果”。这个通信方向主要是服务端到客户端的单向流式返回，并不需要像在线聊天那样保持复杂的双向实时通信。所以 SSE 更轻量，协议更简单，也天然适合大模型逐 token 返回的场景。

SSE 适合：

```
用户提问
  ↓
服务端调用大模型
  ↓
模型边生成边返回
  ↓
前端逐步展示回答
```

WebSocket 更适合：

```
在线聊天
协同编辑
多人游戏
实时行情
双向高频通信
```

面试可以补一句：

> 如果是 B/C 端即时聊天，我会更倾向 WebSocket；但 RAG 问答的流式回答是典型的单向服务端推送，所以 SSE 更合适。

#### 1. SSE 和 WebSocket 的区别

可以从几个维度说：

|对比点|SSE|WebSocket|
|---|---|---|
|通信方向|服务端到客户端单向推送|客户端和服务端双向通信|
|协议|基于 HTTP|独立的 WebSocket 协议|
|使用复杂度|简单|相对复杂|
|自动重连|原生 EventSource 支持|需要自己实现|
|数据格式|文本流，常用 `text/event-stream`|文本或二进制|
|适合场景|大模型流式输出、通知、日志推送|IM、实时协作、游戏、行情|
|请求方式|原生 EventSource 主要是 GET|握手后双向发送|
|中断控制|fetch stream + AbortController 更灵活|主动 close 连接|

面试回答：

> SSE 是基于 HTTP 的单向服务端推送，浏览器原生 EventSource 支持自动重连，使用成本低；WebSocket 是全双工通信，适合客户端和服务端都需要高频发送消息的场景。我的 RAG 项目里，用户问题通过普通 HTTP 请求发送，模型回答由服务端持续返回，通信模式更符合 SSE，所以没有必要引入 WebSocket 的连接维护、心跳和重连复杂度。

---

# 3. SSE 的请求和响应格式是什么？

## 请求

如果用原生 `EventSource`：

```
const eventSource = new EventSource('/api/chat/stream?conversationId=1&question=xxx')
```

它本质是一个 GET 请求。

## 响应头

服务端一般返回：

```
Content-Type: text/event-streamCache-Control: no-cacheConnection: keep-alive
```

## 响应体格式

SSE 是一段一段文本事件，每条消息以空行分隔：

```
data: 你好data: ，我是data: AI 助手data: [DONE]
```

也可以带事件名：

```
event: messagedata: {"content":"你好"}event: donedata: [DONE]
```

还可以带 id：

```
id: 123event: messagedata: {"content":"第一段"}
```

面试回答：

> SSE 的响应类型是 `text/event-stream`。服务端会持续写入文本事件，每条事件通常由 `data:` 开头，以空行作为结束。前端收到每个 data 后，把内容追加到当前消息里。当收到 `[DONE]` 或 done 事件时，说明本次流式回答结束。

---

# 4. EventSource 只能发 GET 请求，如果需要 POST 参数怎么办？

这是很容易被追问的点。

原生 `EventSource` 的限制是：

```
只能 GET不能直接带 request body自定义 header 不方便AbortController 控制不灵活
```

解决方案有三种。

## 方案一：参数放 query

适合简单参数：

```
const url = `/api/chat/stream?conversationId=${id}&question=${encodeURIComponent(question)}`const es = new EventSource(url)
```

缺点：问题太长不适合放 URL，隐私和长度都有问题。

## 方案二：先 POST 创建任务，再 EventSource 监听

```
POST /api/chat  ↓返回 taskId  ↓GET /api/chat/stream?taskId=xxx
```

优点：兼容 EventSource，又能传复杂参数。

## 方案三：不用原生 EventSource，用 fetch 读取 stream

```
const response = await fetch('/api/chat/stream', {  method: 'POST',  headers: {    'Content-Type': 'application/json'  },  body: JSON.stringify({    conversationId,    question,    topK,    temperature  })})
```

然后用 `ReadableStream` 读取流。

面试回答建议：

> 如果参数很简单，可以放在 query 里；如果是复杂参数，比如问题内容、知识库 id、模型参数、检索配置，我更推荐用 `fetch` 发送 POST，然后读取 response body 的 stream。这样可以带 body、headers，也可以结合 AbortController 实现中断生成。

---

# 5. 你是用原生 EventSource，还是 fetch 读取 stream？

你的项目可以这样回答会更稳：

> 我更倾向使用 fetch 读取 stream，而不是原生 EventSource。因为问答请求通常需要 POST body，比如 question、conversationId、knowledgeBaseId、topK、temperature 等参数；同时我还需要用 AbortController 实现中断生成。原生 EventSource 对 POST 和中断控制不够灵活，所以我用 fetch + ReadableStream 来处理。

示例：

```
const controller = new AbortController()const response = await fetch('/api/chat/stream', {  method: 'POST',  headers: {    'Content-Type': 'application/json'  },  body: JSON.stringify({    conversationId,    question  }),  signal: controller.signal})const reader = response.body!.getReader()const decoder = new TextDecoder('utf-8')while (true) {  const { done, value } = await reader.read()  if (done) break  const chunk = decoder.decode(value, { stream: true })  // 解析 chunk，追加到当前回答}
```

---

# 6. 如何实现模型回复逐字显示？

核心流程：

```
读取流  ↓解析服务端返回的 chunk  ↓拿到 content  ↓追加到当前 assistant 消息  ↓页面响应式更新
```

简单版：

```
message.content += delta
```

但真实项目不要每个 token 都直接更新 DOM，可以配合 buffer + rAF：

```
let buffer = ''let rafId: number | null = nullfunction appendDelta(delta: string) {  buffer += delta  if (rafId === null) {    rafId = requestAnimationFrame(() => {      currentMessage.content += buffer      buffer = ''      rafId = null    })  }}
```

面试回答：

> 服务端每返回一段 delta，前端就把这段内容追加到当前 assistant 消息中。为了避免每个 token 都触发一次响应式更新，我会先把内容放到 buffer，再通过 requestAnimationFrame 合并一帧内的 token，批量更新消息内容，这样能减少页面抖动和渲染压力。

---

# 7. 如何处理中断生成？

中断生成分两层：

## 前端中断

使用 `AbortController` 取消 fetch：

```
controller.abort()
```

## 服务端中断

前端中断后，服务端应该停止继续调用模型或停止继续写流。

前端状态要更新：

```
generating → aborted停止 loading保留已生成内容显示“已停止生成”
```

面试回答：

> 中断生成时，我会调用 AbortController 的 `abort()` 取消当前 fetch 请求。前端会把当前消息状态从 generating 改成 aborted，并保留已经生成的内容。服务端也应该感知连接关闭，停止继续请求大模型或停止写入流，避免浪费模型调用资源。

---

# 8. AbortController 在这里怎么用？

用法：

```
let controller: AbortController | null = nullasync function sendMessage(question: string) {  controller = new AbortController()  try {    const response = await fetch('/api/chat/stream', {      method: 'POST',      body: JSON.stringify({ question }),      signal: controller.signal    })    const reader = response.body!.getReader()    while (true) {      const { done, value } = await reader.read()      if (done) break      // 处理流式内容    }  } catch (err) {    if ((err as DOMException).name === 'AbortError') {      console.log('用户主动中断')      return    }    throw err  }}function stopGenerate() {  controller?.abort()}
```

面试回答：

> AbortController 主要用来取消当前流式请求。发送消息时创建 controller，把 `controller.signal` 传给 fetch。用户点击停止生成时调用 `controller.abort()`，fetch 会抛出 AbortError，我会单独识别这个错误，不把它当成普通网络失败，而是更新为用户主动停止状态。

---

# 9. 页面切换后仍然持续接收结果，你是怎么做到的？

关键点：**流式请求不能绑死在页面组件生命周期里。**

如果把请求写在某个页面组件里：

```
页面卸载  ↓组件销毁  ↓请求状态丢失
```

更好的方式是放到全局 store / composable / service 里。

```
ChatStreamManagerPinia conversationStore全局 activeRequestMap
```

结构可以是：

```
const activeStreams = new Map<string, {  controller: AbortController  messageId: string  conversationId: string  status: 'generating' | 'done' | 'aborted' | 'error'}>()
```

面试回答：

> 我不会把 SSE 请求只放在页面组件内部，而是抽到全局的流式任务管理模块，配合 Pinia 存储会话状态、消息列表和当前生成状态。页面组件只是订阅 store 数据。这样路由切换时，组件可以销毁，但请求和消息状态仍然存在于全局 store 里，流式内容继续写入对应 conversationId 和 messageId。用户切回页面时，可以继续看到最新内容。

要补充边界：

> 这种方式能解决 SPA 内部路由切换。如果用户刷新浏览器页面，前端内存里的 fetch 连接会断掉，这时需要服务端 taskId 和断点续传/重新拉取历史消息来恢复。

---

# 10. 如果用户连续发送多个问题，如何避免流式内容串台？

核心：**每个请求绑定唯一 conversationId、messageId、requestId。**

不要只维护一个全局 `currentAnswer`。

错误做法：

```
currentAnswer += delta
```

正确做法：

```
messages[messageId].content += delta
```

可以设计：

```
{  requestId: 'req_001',  conversationId: 'conv_001',  userMessageId: 'msg_user_001',  assistantMessageId: 'msg_ai_001'}
```

每次收到 chunk 时：

```
appendToMessage({  conversationId,  messageId: assistantMessageId,  delta})
```

面试回答：

> 我会给每次发送生成唯一 requestId 和 assistantMessageId，服务端返回的 chunk 也要能关联到这次请求。前端追加内容时，不写到一个全局 currentAnswer，而是根据 conversationId 和 messageId 精准更新对应消息。这样即使用户连续发送多个问题，或者多个会话同时生成，也不会把 A 问题的 token 追加到 B 问题里。

也可以补充策略：

> 产品上也可以限制同一会话同一时间只允许一个问题生成，新的问题要么排队，要么先中断上一个生成。

---

# 11. SSE 断开后是否重连？

要分情况。

## 原生 EventSource

浏览器默认会自动重连。

但要注意：

```
自动重连适合通知流不一定适合大模型回答因为可能导致重复 token 或上下文错乱
```

## fetch stream

不会自动重连，需要自己实现。

大模型流式回答断开后，重连比较复杂，因为要知道：

```
已经生成到哪里服务端是否还在生成是否支持 taskId是否支持从某个 offset 继续读取
```

面试回答：

> 如果用原生 EventSource，它有自动重连能力；但在大模型流式回答场景里，我不会无脑自动重连，因为可能造成重复输出或内容不连续。如果是 fetch stream，就没有自动重连，需要自己处理。更稳妥的方案是服务端为每次生成维护 taskId，前端断线后可以根据 taskId 查询任务状态或拉取已生成内容。如果服务端不支持恢复，就提示生成中断，让用户重试。

---

# 12. 服务端返回 `[DONE]` 时，前端如何处理？

`[DONE]` 表示本次流式输出完成。

前端要做几件事：

```
停止读取流把消息状态改为 done清空 buffer关闭 reader取消 loading保存完整消息触发最终 Markdown 渲染更新历史会话
```

示例：

```
if (data === '[DONE]') {  flushBuffer()  updateMessage(messageId, {    status: 'done'  })  saveMessageToHistory(messageId)  break}
```

面试回答：

> 收到 `[DONE]` 后，我会认为本次模型回复已经完成。前端会先 flush 掉剩余 buffer，然后把当前 assistant 消息状态改成 done，关闭 loading，清理 controller 和 activeStream 记录，并把完整内容写入历史会话。如果流式阶段 Markdown 是降频或半结构渲染，done 后会再做一次完整 marked 渲染，保证最终展示正确。

---

# 13. 流式输出中 Markdown 没闭合时，如何渲染？

这是你前面问过的重点。

流式 Markdown 常见问题：

```
代码块 ``` 没闭合表格还没输出完列表还在生成链接语法没闭合
```

不推荐每个 token 都：

```
html = marked.parse(fullText)
```

推荐：

```
稳定内容：marked 渲染正在输出的尾部：纯文本展示[DONE] 后：完整 marked 渲染
```

面试回答：

> 流式过程中 Markdown 可能是不完整的，比如代码块还没闭合、表格还没生成完。如果每个 token 都全量 marked 解析，可能出现结构闪烁和性能问题。所以我会把内容分成稳定区和流式尾部，稳定区低频调用 marked 渲染，尾部先用纯文本展示。收到 `[DONE]` 后，再对完整内容做一次 marked 渲染。

代码思路：

```
const { stable, tail } = splitStableMarkdown(rawText)renderedHtml.value = marked.parse(stable)streamingTail.value = tail
```

---

# 14. 流式消息如何保存到历史会话？

可以分两阶段保存。

## 阶段一：生成中

前端 store 实时维护：

```
{  id: 'msg_ai_001',  role: 'assistant',  content: '正在生成的内容...',  status: 'generating',  createdAt: Date.now()}
```

## 阶段二：生成完成

收到 `[DONE]` 后保存完整内容：

```
conversationIdmessageIdrolecontentmetadata引用来源生成状态 donecreatedAtupdatedAt
```

面试回答：

> 流式过程中，消息会先以 generating 状态存在 Pinia 的消息列表里，每次收到 delta 都追加到这条 assistant 消息。收到 `[DONE]` 后，把状态改成 done，并把完整内容保存到历史会话。如果中途失败或用户中断，也会保存当前已生成内容，但状态标记为 error 或 aborted，方便用户继续查看或重新生成。

可以补充：

> 如果需要更可靠，服务端也应该边生成边记录或者在结束时落库，避免前端刷新后丢失生成结果。

---

# 15. 如何处理网络中断、模型超时、接口报错？

要区分不同错误类型。

## 网络中断

表现：

```
fetch 报错reader.read 失败连接突然断开
```

处理：

```
消息状态改为 error保留已生成内容提示网络中断允许重试 / 重新生成
```

## 用户主动中断

`AbortError`

处理：

```
状态改为 aborted不展示红色错误提示“已停止生成”
```

## 模型超时

处理：

```
前端设置超时计时器服务端也设置模型调用超时超时后 abort提示模型响应超时
```

## 接口报错

服务端可以返回 error event：

```
event: errordata: {"message":"模型调用失败"}
```

前端解析后：

```
状态改为 error展示错误原因允许重试
```

面试回答：

> 我会把错误分成用户主动中断、网络异常、模型超时和服务端业务错误。用户主动中断对应 AbortError，不当成失败；网络中断和模型超时会把消息状态改成 error，但保留已生成内容；服务端如果返回 error event，就展示具体错误原因。所有异常都会清理 loading、controller 和 activeStream，避免页面一直处于生成中。

---

# 16. 一段完整的 fetch stream 示例

面试时不用完整写，但你可以理解这个流程：

```
async function streamChat(params: {  conversationId: string  messageId: string  question: string}) {  const controller = new AbortController()  activeStreams.set(params.messageId, {    controller,    conversationId: params.conversationId  })  try {    const response = await fetch('/api/chat/stream', {      method: 'POST',      headers: {        'Content-Type': 'application/json'      },      body: JSON.stringify(params),      signal: controller.signal    })    if (!response.ok || !response.body) {      throw new Error('流式请求失败')    }    const reader = response.body.getReader()    const decoder = new TextDecoder('utf-8')    let buffer = ''    while (true) {      const { done, value } = await reader.read()      if (done) break      buffer += decoder.decode(value, { stream: true })      const lines = buffer.split('\n')      buffer = lines.pop() || ''      for (const line of lines) {        if (!line.startsWith('data:')) continue        const data = line.replace(/^data:\s*/, '')        if (data === '[DONE]') {          flushMessage(params.messageId)          markMessageDone(params.messageId)          activeStreams.delete(params.messageId)          return        }        const parsed = JSON.parse(data)        appendDelta(params.messageId, parsed.content)      }    }  } catch (err) {    if ((err as DOMException).name === 'AbortError') {      markMessageAborted(params.messageId)    } else {      markMessageError(params.messageId, err)    }  } finally {    activeStreams.delete(params.messageId)  }}
```

---

# 17. 这组问题的完整面试回答

你可以直接这样回答：

> 我在 RAG 问答里选择 SSE，是因为这个场景主要是用户发送问题后，服务端持续向客户端推送模型生成内容，属于单向流式返回，不需要 WebSocket 那种全双工通信。WebSocket 更适合在线聊天、协同编辑这类双向实时场景，而 SSE 基于 HTTP，更轻量，也更符合大模型流式输出。
> 
> 原生 SSE 的响应类型是 `text/event-stream`，服务端通过 `data:` 持续返回内容，以空行分隔事件，最后返回 `[DONE]` 表示生成结束。不过原生 EventSource 只能发 GET，不方便传复杂参数和中断控制，所以我更倾向用 fetch 发 POST，然后通过 `ReadableStream` 读取服务端返回的流。这样可以带 question、conversationId、topK、temperature 等参数，也能用 AbortController 实现停止生成。
> 
> 前端收到每段 delta 后，不会每个 token 都直接更新 DOM，而是先放到 buffer 中，通过 requestAnimationFrame 合并一帧内的内容，再追加到当前 assistant 消息。消息状态和内容通过 Pinia 管理，每次请求都会绑定 conversationId、messageId 和 requestId，避免连续提问时内容串台。
> 
> 页面切换后仍然接收结果，是因为流式请求不绑定在页面组件内部，而是放在全局的 stream manager 或 Pinia action 里。页面只是订阅 store 数据，所以组件卸载后，请求仍然可以继续写入对应消息。
> 
> 收到 `[DONE]` 后，会 flush 剩余内容，把消息状态改为 done，清理 controller 和 activeStream，并保存完整历史消息。流式 Markdown 渲染时，我会避免每个 token 都全量 marked 解析，而是稳定内容低频渲染，未闭合的尾部先用纯文本展示，最后完成后再完整渲染一次。
> 
> 异常处理上，我会区分用户主动中断、网络中断、模型超时和服务端报错。AbortError 视为用户停止，不算失败；网络和模型错误会把消息状态标记为 error，保留已生成内容，并提供重试或重新生成入口。

## 7. marked Markdown 渲染

1. 为什么选择 marked？
2. marked 和 markdown-it 有什么区别？
3. 大模型返回 Markdown 时，可能有哪些安全风险？
4. 如何防止 XSS？
5. 是否使用 DOMPurify？
6. 代码块如何高亮？
7. 流式输出时 Markdown 不完整，如何避免页面闪烁？
8. 表格、引用、列表是否支持？
9. 用户输入内容是否也经过 Markdown 渲染？
10. 如果模型返回 HTML 标签，你怎么处理？

## 8. ResizeObserver + 自动滚动

1. 为什么需要 ResizeObserver？
2. 为什么不用简单的 scrollTop = scrollHeight？
3. 流式输出过程中，消息气泡高度为什么会变化？
4. 如何判断用户是否正在手动向上查看历史消息？
5. 如果用户已经滚到上面，是否还要自动吸底？
6. ResizeObserver 回调频繁触发会不会影响性能？
7. 如何防抖或节流？
8. 自动滚动时如何减少抖动？
9. 图片或代码块加载后高度变化如何处理？
10. ResizeObserver 有什么兼容性问题？

## 9. 分片上传、秒传、断点续传

1. 为什么需要分片上传？
2. 分片大小如何选择？
3. 文件 hash 是怎么计算的？
4. 大文件计算 hash 会不会阻塞主线程？
5. 秒传的原理是什么？
6. 断点续传的流程是什么？
7. 服务端如何记录已上传分片？
8. 前端如何查询已上传分片？
9. 分片上传失败如何重试？
10. 多分片并发上传时，如何控制并发数？
11. 上传进度如何计算？
12. 如果某个分片重复上传，服务端如何处理？
13. 文件最终合并由谁触发？
14. 如何校验合并后的文件完整性？
15. 如果用户刷新页面，如何恢复上传任务？

## 10. requestAnimationFrame 批处理渲染

1. 为什么流式输出会带来渲染压力？
2. requestAnimationFrame 的执行时机是什么？
3. 它和 setTimeout、requestIdleCallback 有什么区别？
4. 你如何用 rAF 批处理消息更新？
5. 如果每个 token 都更新一次 Pinia，会有什么问题？
6. rAF 能解决所有性能问题吗？
7. 流式输出很快时，如何合并多段文本？
8. 页面不可见时 rAF 会怎样？
9. 如果用户切换标签页，流式更新如何处理？
10. Vue 的响应式更新机制和 rAF 如何配合？