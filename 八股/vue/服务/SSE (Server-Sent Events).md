SSE 全称 **Server-Sent Events（服务器发送事件）**，是 HTML5 标准原生支持的、基于 HTTP 协议的**轻量级单向实时通信技术**。它允许客户端与服务器建立一次持久化的 HTTP 长连接后，由服务器持续、主动地向客户端推送流式数据，客户端通过浏览器原生的 `EventSource` API 监听并处理消息，是前端实现服务器推送的最优方案之一，也是当前 AI 大模型流式对话、实时通知等场景的核心技术。
## 一、核心工作原理

SSE 完全基于标准 HTTP 协议实现，无需额外的协议升级或第三方依赖，核心工作流程如下：

1. **连接建立**：客户端通过 `EventSource` API 向服务器发起一个标准的 HTTP GET 请求，声明自己希望接收事件流；
2. **长连接保持**：服务器收到请求后，返回 `Content-Type: text/event-stream` 格式的响应头，同时设置 `Connection: keep-alive` 保持 TCP 连接不断开，不关闭响应流；
3. **持续数据推送**：服务器可以在任意时刻，通过这个已建立的长连接，向客户端推送符合 SSE 格式规范的文本数据；
4. **客户端自动解析**：浏览器的 `EventSource` 对象会持续监听响应流，自动解析推送的消息，并触发对应的事件回调；
5. **断线自动重连**：如果连接意外中断，浏览器会自动按照默认 / 自定义的间隔尝试重新建立连接，无需开发者手动实现。

## 二、SSE 核心特性

1. **原生自动重连**：内置断线重连机制，连接异常中断时浏览器会自动重试，还可通过服务端自定义重连间隔，无需手动开发；
2. **断点续传能力**：支持通过 `id` 字段标记每一条消息，重连时浏览器会自动带上 `Last-Event-ID` 请求头，服务器可从断点继续推送，避免消息丢失；
3. **自定义事件机制**：不仅支持默认的 `message` 事件，还可通过 `event` 字段自定义事件类型，实现多类型消息的分类监听与处理；
4. **纯 HTTP 协议**：无需像 WebSocket 那样做协议升级，完全兼容现有 HTTP 基础设施（CDN、代理、网关、鉴权体系），无防火墙穿透问题；
5. **浏览器原生支持**：所有现代浏览器（Chrome、Firefox、Safari、Edge）均原生支持，无需引入任何第三方 SDK，代码量极少；
6. **文本流友好**：默认采用 UTF-8 文本编码，对 JSON 字符串、纯文本流式内容的传输效率极高。

## 三、SSE 与其他实时通信方案对比

前端主流的实时通信方案包括短轮询、长轮询、SSE、WebSocket，核心差异与选型参考如下：

|特性|SSE|WebSocket|长轮询|短轮询|
|---|---|---|---|---|
|通信方向|服务器→客户端 单向|客户端↔服务器 全双工双向|客户端→服务器 单向请求 - 响应|客户端→服务器 单向请求 - 响应|
|协议基础|标准 HTTP/HTTPS|独立 WebSocket 协议（需 HTTP 握手升级）|标准 HTTP/HTTPS|标准 HTTP/HTTPS|
|实现复杂度|极低，浏览器原生封装|中等，需手动实现重连、心跳、断连处理|中等，需手动管理请求生命周期|极低，定时器 + 请求即可|
|自动重连|原生内置支持|需手动开发实现|无，响应后需手动发起新请求|无，完全依赖定时器|
|数据格式|原生仅支持 UTF-8 文本（二进制需转 Base64）|原生支持文本、二进制数据|任意 HTTP 兼容格式|任意 HTTP 兼容格式|
|带宽开销|极低，仅一次 HTTP 头部，后续无冗余开销|极低，帧格式传输，无 HTTP 头部冗余|中等，每次请求都带完整 HTTP 头部|极高，频繁请求带大量重复头部|
|实时性|高，服务器推送无延迟|极高，毫秒级双向传输|中，依赖服务器响应后重新发起请求|低，完全依赖轮询间隔|
|浏览器并发限制|HTTP/1.1 下同一域名最多 6 个并发连接|无浏览器并发限制|受浏览器 HTTP 并发数限制|受浏览器 HTTP 并发数限制|
|核心适用场景|AI 流式输出、实时通知、日志监控、行情推送|在线聊天室、协同编辑、实时游戏、双向互动|低频次消息通知、兼容性要求极高的场景|极简场景、对实时性无要求的场景|
### 核心选型结论

- 仅需**服务器向客户端单向推送数据**，优先选 SSE：开发成本最低、维护最简单、原生能力覆盖绝大多数场景；
- 需要**频繁双向通信**（如聊天、游戏），必须选 WebSocket。

## 四、核心规范：消息格式与客户端 API

### （一）服务端 SSE 消息格式规范

服务器推送的消息必须严格遵循 UTF-8 文本格式，每条消息以**两个换行符 `\n\n`** 作为结束标记，单条消息可包含以下 5 个核心字段，每行一个字段，以 `字段名: 值` 的格式书写：

|字段|作用|必选|
|---|---|---|
|`data`|消息体内容，核心字段，可多行书写，最终会拼接成完整字符串|是|
|`event`|自定义事件名称，客户端需通过 `addEventListener` 监听对应事件，不写则触发默认 `message` 事件|否|
|`id`|消息唯一 ID，重连时浏览器会自动通过 `Last-Event-ID` 请求头带给服务器，用于断点续传|否|
|`retry`|自定义断线重连的时间间隔，单位为毫秒，浏览器默认重连间隔为 3000ms|否|
|`: 注释内容`|以冒号开头的行为注释，客户端不会触发任何事件，常用于心跳保活|否|

#### 合法消息示例

```text
# 1. 最简单的默认消息
data: 你好，这是一条SSE消息\n\n

# 2. 多行JSON数据
data: {
data:   "name": "张三",
data:   "age": 25
data: }\n\n

# 3. 自定义事件+ID+重连间隔
retry: 5000
id: msg_1001
event: notice
data: 您有一条新的系统通知\n\n

# 4. 心跳注释（不触发事件，仅保活）
: heartbeat\n\n
```

### （二）客户端 EventSource API

浏览器原生提供 `EventSource` 接口，用于创建和管理 SSE 连接，核心用法如下：
#### 1. 创建连接

```javascript
// 同源连接
const eventSource = new EventSource('/api/stream');

// 跨域连接（需服务端配置CORS），第二个参数开启跨域Cookie携带
const eventSource = new EventSource('https://api.example.com/stream', {
  withCredentials: true
});
```

#### 2. 核心事件监听

```javascript
// 1. 连接成功建立时触发
eventSource.onopen = () => {
  console.log('SSE 连接已建立', eventSource.readyState);
};

// 2. 收到默认消息（无event字段的消息）时触发
eventSource.onmessage = (event) => {
  console.log('收到消息：', event.data);
  console.log('消息ID：', event.lastEventId);
};

// 3. 监听自定义事件（对应服务端event字段）
eventSource.addEventListener('notice', (event) => {
  console.log('收到系统通知：', event.data);
});

eventSource.addEventListener('ai-stream', (event) => {
  // AI流式输出，逐字拼接内容
  document.getElementById('ai-content').innerText += event.data;
});

// 4. 连接发生错误时触发（网络中断、服务器关闭连接等）
eventSource.onerror = (error) => {
  console.error('SSE 连接异常', error);
  // 连接关闭时，手动终止重连
  if (eventSource.readyState === EventSource.CLOSED) {
    console.log('连接已关闭');
  }
};
```

#### 3. 核心属性与方法

|属性 / 方法|作用|
|---|---|
|`eventSource.readyState`|连接状态：0（CONNECTING 连接中）、1（OPEN 已连接）、2（CLOSED 已关闭）|
|`eventSource.url`|连接的目标地址|
|`eventSource.withCredentials`|是否跨域携带 Cookie|
|`eventSource.close()`|手动永久关闭连接，关闭后浏览器不会再自动重连|

## 五、完整可运行的前后端示例

### （一）前端代码（原生 HTML+JS）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>SSE 流式示例</title>
</head>
<body>
  <h3>AI 流式回复：</h3>
  <div id="ai-content" style="width: 600px; line-height: 1.8;"></div>
  <button id="close-btn">关闭连接</button>

  <script>
    const contentBox = document.getElementById('ai-content');
    const closeBtn = document.getElementById('close-btn');

    // 创建SSE连接
    const eventSource = new EventSource('/api/ai-stream');

    // 监听流式消息
    eventSource.addEventListener('ai-stream', (event) => {
      contentBox.innerText += event.data;
    });

    // 监听结束事件
    eventSource.addEventListener('stream-end', () => {
      contentBox.innerText += '\n\n--- 回复已结束 ---';
      eventSource.close(); // 手动关闭连接
    });

    // 错误处理
    eventSource.onerror = (err) => {
      console.error('连接异常', err);
      if (eventSource.readyState === EventSource.CLOSED) {
        contentBox.innerText += '\n\n连接已断开';
      }
    };

    // 手动关闭连接
    closeBtn.onclick = () => {
      eventSource.close();
      contentBox.innerText += '\n\n用户手动关闭了连接';
    };
  </script>
</body>
</html>
```

### （二）服务端代码（Node.js + Express）

```javascript
const express = require('express');
const app = express();
const port = 3000;

// 静态资源托管，前端HTML放在public目录
app.use(express.static('public'));

// SSE 流式接口
app.get('/api/ai-stream', (req, res) => {
  // 1. 设置SSE必需的响应头
  res.writeHead(200, {
    'Content-Type': 'text/event-stream; charset=utf-8',
    'Connection': 'keep-alive',
    'Cache-Control': 'no-cache',
    'Access-Control-Allow-Origin': '*', // 跨域配置，生产环境替换为具体域名
  });

  // 2. 模拟AI流式输出，逐字推送内容
  const content = '你好，我是豆包AI，很高兴为你服务。SSE技术是实现流式输出的核心方案，它轻量、易用，非常适合AI对话场景。';
  let index = 0;

  // 每50ms推送一个字
  const timer = setInterval(() => {
    if (index >= content.length) {
      // 推送结束事件
      res.write('event: stream-end\n');
      res.write('data: done\n\n');
      clearInterval(timer);
      res.end(); // 关闭响应流
      return;
    }

    // 推送单字内容
    res.write('event: ai-stream\n');
    res.write(`data: ${content[index]}\n\n`);
    index++;
  }, 50);

  // 3. 客户端断开连接时，清理资源
  req.on('close', () => {
    clearInterval(timer);
    console.log('客户端断开连接，资源已清理');
  });
});

app.listen(port, () => {
  console.log(`服务运行在 http://localhost:${port}`);
});
```

## 六、核心适用场景

SSE 是**单向服务器推送**场景的最优解，典型适用场景包括：

1. **AI 大模型流式对话**：ChatGPT、豆包等 AI 产品的逐字流式回复，是当前 SSE 最主流的应用场景；
2. **系统实时通知 / 公告**：站内信、订单状态变更、告警通知、审批提醒等；
3. **实时数据看板**：服务器监控指标、股票 / 加密货币行情、赛事比分、数据大屏实时刷新；
4. **日志实时流输出**：前端实时展示服务器日志、构建进度、任务执行进度；
5. **实时内容推送**：新闻资讯更新、社交媒体动态、直播弹幕等。

## 七、优缺点分析

### 核心优点

1. **开发成本极低**：浏览器原生支持，前后端代码量极少，无需引入第三方依赖，学习成本低；
2. **原生能力完善**：内置自动重连、断点续传、自定义事件，无需手动开发复杂的连接管理逻辑；
3. **兼容性与穿透性好**：完全基于 HTTP 协议，兼容所有现有网关、CDN、防火墙，无协议兼容问题；
4. **轻量低开销**：仅一次 HTTP 握手，后续无冗余头部开销，带宽占用远低于轮询方案；
5. **调试友好**：可直接在浏览器 Network 面板的 EventStream 标签中查看实时推送的消息，调试成本极低。

### 核心缺点

1. **单向通信限制**：仅支持服务器向客户端推送数据，客户端要向服务器发送消息，必须单独发起新的 HTTP 请求；
2. **浏览器并发限制**：HTTP/1.1 环境下，浏览器对**同一域名**的 SSE 连接最多只允许同时存在 6 个，超出会被阻塞；HTTP/2 环境下默认无此限制；
3. **默认仅支持文本数据**：原生不支持二进制数据传输，需转 Base64 编码后才能传输，额外增加开销；
4. **IE 浏览器完全不兼容**：仅支持现代浏览器，IE 全版本不支持（目前绝大多数场景可忽略）。

## 八、生产环境最佳实践与避坑指南

### 1. 必做：关闭代理 / 网关缓冲

Nginx、CDN 等反向代理默认会缓冲响应流，导致客户端无法实时收到消息，必须关闭缓冲：

- 服务端响应头添加：`X-Accel-Buffering: no`
- Nginx 配置补充：

```nginx
location /api/stream {
  proxy_pass http://你的后端服务;
  proxy_buffering off; # 核心：关闭代理缓冲
  proxy_cache off;
  proxy_http_version 1.1;
  proxy_set_header Connection keep-alive;
  # 延长超时时间，默认60秒会断开连接
  proxy_read_timeout 86400s;
  proxy_send_timeout 86400s;
}
```

### 2. 必做：心跳保活机制

防火墙、代理服务器会自动关闭长时间无数据传输的长连接，必须定期发送心跳包维持连接：

- 服务端每隔 15-30 秒发送一条注释行 `: heartbeat\n\n`，不会触发客户端事件，仅用于保活；
- 客户端可监听超时，长时间未收到消息时主动重连。

### 3. 跨域与鉴权方案

原生 `EventSource` 无法自定义请求头（如 `Authorization`），推荐两种鉴权方案：

- 方案 1：通过 Cookie 携带身份凭证，配合 `withCredentials: true` 开启跨域 Cookie 传输；
- 方案 2：将 Token 放在 URL 的 query 参数中，服务端解析并校验，HTTPS 环境下参数同样加密安全。

### 4. 解决浏览器并发限制

- 优先启用 HTTP/2，HTTP/2 支持多路复用，同一域名的多个 SSE 连接共享一个 TCP 连接，无 6 个并发的限制；
- 合并同页面的多个 SSE 流，通过一个连接推送多类型消息，用自定义事件区分，减少连接数。

### 5. 集群部署适配

多实例集群部署时，需保证同一个客户端的 SSE 连接始终命中同一台服务器，否则会出现推送失败：

- 简单方案：Nginx 配置 `ip_hash`，通过 IP 哈希实现粘滞会话；
- 稳健方案：通过 Redis 发布订阅（Pub/Sub）实现多实例消息广播，无论客户端连到哪台实例都能收到消息。

### 6. 资源释放与异常处理

- 服务端必须监听客户端的 `close` 事件，及时清理定时器、数据库连接等资源，避免内存泄漏；
- 客户端页面卸载时，必须调用 `eventSource.close()` 主动关闭连接，避免浏览器自动重连；
- 自定义重连策略，默认的无限重连可能给服务器造成压力，可实现指数退避重连（失败后间隔翻倍）。