Mock 数据是指在**开发、测试阶段**，通过模拟真实 API 的请求 / 响应流程，返回预设数据的技术。它的核心价值是**解耦依赖**（如后端未就绪、外部 API 限制）、**提升效率**（不阻塞开发进度）、**覆盖场景**（模拟错误、延迟、边界数据）。

## 一、为什么需要 Mock 数据？

以下是 Mock 数据的核心应用场景：

1. **前后端分离开发**：后端接口未完成时，前端可基于 Mock 数据并行开发，无需等待。
2. **测试环境隔离**：单元测试、E2E 测试不依赖真实服务，避免数据污染，且能稳定复现场景。
3. **复杂场景模拟**：轻松模拟网络延迟（`delay: 1000ms`）、错误状态码（401/404/500）、边界数据（空列表、超长字符串）。
4. **外部依赖解耦**：依赖第三方 API（如支付、地图、短信）时，避免调用成本、频率限制，且无需网络。

## 二、Mock 数据的实现方式（按拦截层级分类）

Mock 工具的核心差异在于**请求拦截的层级**—— 层级越接近真实网络，调试体验和场景还原度越高。以下从 “代码层” 到 “网络层” 逐一介绍：

### 1. 代码层 Mock（最底层，与业务耦合）

直接在业务代码中通过环境变量或条件判断返回 Mock 数据，或在请求库内部拦截。

#### （1）手动硬编码 Mock

- **原理**：在 API 调用函数中判断环境，开发时直接返回预设数据。
- **示例**：
    
```javascript
    // api/user.js
    export const fetchUser = async () => {
      // 开发环境直接返回 Mock 数据
      if (process.env.NODE_ENV === 'development') {
        return { id: 1, name: '张三', email: 'zhangsan@example.com' };
      }
      // 生产环境发送真实请求
      return fetch('/api/user').then(res => res.json());
    };
```
    
- **优点**：零依赖、实现极快，适合临时验证。
- **缺点**：与业务代码强耦合，无法模拟网络请求（如延迟、错误码），维护成本高。

#### （2）请求库拦截器（如 `axios-mock-adapter`）

- **原理**：在 axios 库的拦截器层面拦截请求，仅适配 axios 项目。
- **示例**：
    
```javascript
    import axios from 'axios';
    import MockAdapter from 'axios-mock-adapter';
    
    const mock = new MockAdapter(axios);
    // 拦截 GET /api/user，返回 Mock 数据
    mock.onGet('/api/user').reply(200, { id: 1, name: '张三' });
    // 拦截 POST /api/login，根据参数动态返回
    mock.onPost('/api/login').reply(config => {
      const { username } = JSON.parse(config.data);
      return username === 'admin' ? [200, { token: '123' }] : [401, { error: '失败' }];
    });
```
    
- **优点**：与 axios 深度集成，配置简单，支持动态逻辑。
- **缺点**：仅支持 axios，无法拦截 `fetch` 或其他请求库；测试环境需单独配置。

### 2. 浏览器层 Mock（重写浏览器 API）

通过重写 `XMLHttpRequest` 或 `fetch` 拦截请求，仅在浏览器环境生效。

#### 代表工具：Mock.js

- **原理**：重写浏览器的 `XMLHttpRequest` 对象，拦截 XHR 请求（需额外适配 fetch）。
- **核心优势**：内置**丰富的随机数据生成规则**（如 `@cname` 随机中文名、`@integer(1,100)` 随机数字）。
- **示例**：
    
```javascript
    import Mock from 'mockjs';
    
    // 拦截 GET /api/users，返回 10 个随机用户
    Mock.mock('/api/users', 'get', {
      'list|10': [{
        'id|+1': 1,
        'name': '@cname',
        'email': '@email',
      }]
    });
```
    
- **优点**：零额外服务，随机数据生成能力强，适合快速生成模拟数据。
- **缺点**：
    
    - 不原生支持 `fetch`（需手动适配）；
    - **Network 面板看不到被拦截的请求**，调试依赖 console.log；
    - 仅浏览器可用，Node.js 测试环境需单独方案。
    
### 3. 独立服务层 Mock（启动额外服务器）

通过启动一个独立的 HTTP 服务器提供 Mock API，支持真实网络请求。
#### 代表工具：JSON Server

- **原理**：基于一个 JSON 文件快速生成 RESTful API，支持 GET/POST/PUT/DELETE 等 CRUD 操作。
- **示例**：
1. 创建 `db.json` 定义数据：

```json
	{
	  "users": [{"id": 1, "name": "张三"}],
	  "posts": [{"id": 1, "title": "Mock 入门", "userId": 1}]
	}
```

2. 启动服务：
```json
	npx json-server --watch db.json --port 3001
```

3. 访问 API：

- `GET http://localhost:3001/users`：返回用户列表
- `POST http://localhost:3001/users`：添加新用户（自动保存到 `db.json`）

- **优点**：零代码，快速生成完整 REST API；支持真实网络请求，Network 面板可见；数据持久化（自动保存到 JSON 文件）。
- **缺点**：动态逻辑支持弱（仅简单 CRUD，复杂条件判断需写自定义路由）；需额外启动服务，增加开发复杂度。

### 4. 网络层 Mock（最接近真实，推荐主流项目）

在**真正的网络层**拦截请求，浏览器环境通过 Service Worker 实现，Node.js 环境通过网络钩子实现，调试体验和真实请求完全一致。

#### 代表工具：MSW（Mock Service Worker）

这是目前最推荐的 Mock 方案，重点结合你之前提到的 “模拟 fetch 更真实” 和 “造数据” 问题详细讲解。
##### （1）MSW 的核心原理

- **浏览器环境**：注册一个 Service Worker，在请求到达网络前拦截（包括 `fetch`、`XHR`、第三方库请求），返回 Mock 数据。**整个流程和真实请求完全一致**——Network 面板能看到完整的请求 URL、方法、状态码、响应头、响应体。
- **Node.js 环境**：拦截 Node.js 的 `http`/`https` 模块请求，支持 Jest/Vitest 等单元测试框架，实现开发 + 测试环境的统一 Mock。
##### （2）MSW 的核心优势

1. **最真实的调试体验**：Network 面板可见所有请求，和调用真实 API 无区别，无需依赖 console.log。
2. **跨环境复用**：同一套 Mock 规则（handlers）可同时用于浏览器开发和 Node.js 测试。
3. **协议无关**：支持 REST API、GraphQL、WebSocket 等，且不依赖特定请求库（axios、fetch、umi-request 等均可）。
4. **灵活的动态逻辑**：支持根据请求参数、请求体、URL 动态返回数据，模拟延迟、错误码等场景。
##### （3）MSW 的 “造数据” 方案

MSW 本身**不负责生成 Mock 数据**，但可以和任何数据生成工具无缝配合，常见方案有 3 种：

###### 方案 1：简单场景 —— 手写 JSON / 对象

适合接口少、数据结构简单的情况，直接在 MSW 的 handler 中返回手写数据：

```javascript
// src/mocks/handlers.js
import { http, HttpResponse, delay } from 'msw';

export const handlers = [
  http.get('/api/user', async () => {
    await delay(500); // 模拟 500ms 网络延迟
    return HttpResponse.json({
      id: 1,
      name: '张三',
      email: 'zhangsan@example.com',
    });
  }),
];
```

###### 方案 2：复杂随机数据 —— 结合 `@faker-js/faker`

如果需要大量、多样化的 Mock 数据（如测试列表页、随机用户信息），用 **`@faker-js/faker`**（最流行的随机数据生成库）配合 MSW：

1. 安装依赖：
    
```bash
    npm install msw @faker-js/faker --save-dev
```
    
2. 定义 handlers（结合 faker 生成数据）：
    
```javascript
    // src/mocks/handlers.js
    import { http, HttpResponse } from 'msw';
    import { faker } from '@faker-js/faker';
    
    // 生成 10 个随机用户的函数
    const generateRandomUsers = (count = 10) => {
      return Array.from({ length: count }, () => ({
        id: faker.string.uuid(),
        name: faker.person.fullName(),
        email: faker.internet.email(),
        avatar: faker.image.avatar(),
        role: faker.helpers.arrayElement(['admin', 'user', 'guest']),
        createdAt: faker.date.past(),
      }));
    };
    
    export const handlers = [
      // 拦截 GET /api/users，返回随机用户列表
      http.get('/api/users', ({ request }) => {
        const url = new URL(request.url);
        const page = parseInt(url.searchParams.get('page') || '1');
        const pageSize = 10;
        const users = generateRandomUsers(pageSize);
        return HttpResponse.json({
          code: 200,
          data: users,
          pagination: { page, pageSize, total: 100 },
        });
      }),
    
      // 拦截 POST /api/login，根据请求体动态返回
      http.post('/api/login', async ({ request }) => {
        const { username, password } = await request.json();
        if (username === 'admin' && password === '123456') {
          return HttpResponse.json({ token: faker.string.uuid() }, { status: 200 });
        }
        return HttpResponse.json({ error: '用户名或密码错误' }, { status: 401 });
      }),
    ];
```

###### 方案 3：持久化 CRUD—— 结合 `lowdb`

如果需要 Mock 数据支持**增删改查后刷新页面数据不丢失**，用 **`lowdb`**（基于 JSON 文件的轻量级数据库）配合 MSW：

1. 安装依赖：
    
```bash
    npm install msw lowdb @faker-js/faker --save-dev
```
    
2. 初始化 lowdb（`src/mocks/db.js`）：
    
```javascript
    import { Low } from 'lowdb';
    import { JSONFile } from 'lowdb/node';
    import { faker } from '@faker-js/faker';
    
    // 定义数据结构
    const defaultData = { users: [] };
    // 初始化 JSON 文件适配器
    const adapter = new JSONFile('mock-db.json');
    const db = new Low(adapter, defaultData);
    
    // 首次运行时初始化 10 个用户
    const initDB = async () => {
      await db.read();
      if (db.data.users.length === 0) {
        db.data.users = Array.from({ length: 10 }, () => ({
          id: faker.string.uuid(),
          name: faker.person.fullName(),
          email: faker.internet.email(),
        }));
        await db.write();
      }
    };
    initDB();
    
    export default db;
```
    
3. 定义 handlers（结合 lowdb 实现 CRUD）：
    
```javascript
    // src/mocks/handlers.js
    import { http, HttpResponse } from 'msw';
    import db from './db';
    import { faker } from '@faker-js/faker';
    
    export const handlers = [
      // 获取用户列表
      http.get('/api/users', async () => {
        await db.read();
        return HttpResponse.json(db.data.users);
      }),
    
      // 添加用户
      http.post('/api/users', async ({ request }) => {
        const newUser = await request.json();
        newUser.id = faker.string.uuid();
        await db.read();
        db.data.users.push(newUser);
        await db.write();
        return HttpResponse.json(newUser, { status: 201 });
      }),
    
      // 删除用户
      http.delete('/api/users/:id', async ({ params }) => {
        await db.read();
        db.data.users = db.data.users.filter(user => user.id !== params.id);
        await db.write();
        return HttpResponse.json({ message: '删除成功' });
      }),
    ];
```

##### （4）MSW 的完整配置流程

1. **浏览器环境配置**（用于开发调试）：
    
创建 `src/mocks/browser.js`：
    
```javascript
    import { setupWorker } from 'msw/browser';
    import { handlers } from './handlers';
    
    export const worker = setupWorker(...handlers);
```
    
在项目入口文件（如 `main.js`/`index.js`）中启动：

```js
	if (process.env.NODE_ENV === 'development') {
      const { worker } = require('./mocks/browser');
      worker.start({
        onUnhandledRequest: 'bypass', // 未拦截的请求直接放行（不影响真实接口调用）
      });
    }
```

2. **Node.js 环境配置**（用于单元测试）：

创建 `src/mocks/server.js`：

```javascript
    import { setupServer } from 'msw/node';
    import { handlers } from './handlers';
    
    export const server = setupServer(...handlers);
```
    
在测试文件（如 Jest/Vitest）中初始化：

```js
	import { server } from './mocks/server';
    
    beforeAll(() => server.listen()); // 测试前启动
    afterEach(() => server.resetHandlers()); // 每次测试后重置 handlers
    afterAll(() => server.close()); // 测试后关闭
```

## 三、各 Mock 方式对比表

|方式|拦截层级|环境支持|调试体验|数据生成能力|动态逻辑|复用性|学习曲线|
|---|---|---|---|---|---|---|---|
|手动硬编码|代码层|全环境|差|低（手写）|低|低|低|
|axios-mock-adapter|axios 拦截器|浏览器 + Node|中|低（手写）|中|中|低|
|Mock.js|XHR 重写|仅浏览器|差（不可见）|高（内置）|中|中|低|
|JSON Server|独立服务器|全环境|中|中（JSON）|低|中|低|
|MSW|网络层（SW）|浏览器 + Node|极佳（Network 可见）|灵活（可配合任意工具）|高|高|中|
## 四、选型建议

根据不同场景选择合适的 Mock 方案：

1. **临时快速验证**：手动硬编码 Mock，零依赖最快。
2. **axios 项目轻量 Mock**：`axios-mock-adapter`，配置简单。
3. **需要大量随机数据**：`MSW + @faker-js/faker`（推荐），或 `Mock.js`（注意调试缺点）。
4. **快速生成 REST API**：`JSON Server`，零代码实现 CRUD。
5. **主流项目推荐**：**MSW**—— 调试体验最真实，跨环境复用，灵活配合数据工具，适合长期维护的项目。

## 总结

Mock 数据的核心是 **“模拟真实场景，不阻塞开发 / 测试”**。其中 MSW 是目前的主流推荐方案，因为它通过网络层拦截实现了 “最真实” 的 Mock 体验，同时通过 “职责分离”（不负责生成数据，但可配合任意工具）保持了极高的灵活性。