双 Token（Access Token + Refresh Token）是现代 Web/H5/APP 最主流的**无状态身份认证方案**，核心解决了「单 Token 模式下安全与用户体验的根本矛盾」：单 Token 有效期太短会导致用户频繁重新登录，太长则一旦泄露会造成长期安全风险。

## 一、核心概念与设计初衷

### 1. 什么是双 Token

双 Token 是一对成对生成、职责完全分离的令牌：

| 令牌类型                    | 核心职责                                          | 有效期                 | 安全级别 | 使用频率                     |
| ----------------------- | --------------------------------------------- | ------------------- | ---- | ------------------------ |
| **Access Token（访问令牌）**  | 访问所有受保护 API 接口的凭证                             | 极短（通常 15 分钟～2 小时）   | 高    | 极高（几乎每次请求都携带）            |
| **Refresh Token（刷新令牌）** | 仅用于在 Access Token 过期后，**无感获取新的 Access Token** | 很长（通常 7 天～30 天，可配置） | 极高   | 极低（仅 Access Token 过期时使用） |

### 2. 为什么必须用双 Token（单 Token 的致命缺陷）

- **安全缺陷**：单 Token 有效期长（比如 7 天），一旦被 XSS / 中间人攻击窃取，攻击者可在 7 天内完全冒充用户操作，且服务器无法即时吊销
- **体验缺陷**：单 Token 有效期短（比如 15 分钟），用户每 15 分钟就要重新输入账号密码登录，体验极差
- **无状态矛盾**：JWT 等无状态 Token 无法被服务器即时作废，只能等自然过期，进一步放大了长有效期的安全风险

双 Token 的核心设计思想：**用高频率、短有效期的 Access Token 承担日常访问，用低频率、高安全的 Refresh Token 解决续期问题**，既保证了安全，又实现了用户无感登录。

## 二、完整端到端流程（从头到尾）

以下是标准的双 Token 登录全流程，覆盖从用户首次登录到最终登出的所有环节：

### 阶段 1：用户首次登录（获取双 Token）

1. **客户端**：用户在登录页输入账号 + 密码（或验证码 / 第三方授权），提交到服务器登录接口
2. **服务器**：
    
    - 验证账号密码的正确性
    - 生成**Access Token**（携带用户 ID、权限、过期时间等信息）
    - 生成**Refresh Token**（随机字符串，与用户 ID 绑定）
    - 将 Refresh Token 存入数据库 / Redis（关联用户 ID、过期时间、设备信息）
    - 将双 Token 返回给客户端
    
3. **客户端**：
    
    - 存储 Access Token（推荐：内存 / Vuex/Redux，避免持久化）
    - 存储 Refresh Token（**强制：HttpOnly+Secure+SameSite Cookie**，绝对不能存在 localStorage/sessionStorage）
    - 跳转到首页，登录完成

### 阶段 2：正常访问受保护资源

1. **客户端**：每次请求受保护 API 时，在请求头中携带 Access Token
    
```http
    Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```
    
2. **服务器**：
    
    - 验证 Access Token 的签名、有效期、权限
    - 验证通过，返回请求的资源数据
    - 验证失败（签名错误 / 已过期），返回`401 Unauthorized`状态码

### 阶段 3：Access Token 过期，无感刷新

1. **客户端**：收到服务器返回的 401 状态码，判断是 Access Token 过期
2. **客户端**：自动调用`/api/refresh-token`接口，**仅携带 Refresh Token**（通过 Cookie 自动携带，无需手动处理）
3. **服务器**：
    
    - 验证 Refresh Token 的有效性（是否存在于数据库、是否过期、是否属于该用户）
    - 验证通过：
        
        - 生成**新的 Access Token**
        - （最佳实践）生成**新的 Refresh Token**，同时将旧的 Refresh Token 从数据库中删除（防止重复使用）
        - 将新的双 Token 返回给客户端
        
    - 验证失败（Refresh Token 过期 / 不存在 / 被吊销）：返回`403 Forbidden`状态码
    
4. **客户端**：
    
    - 收到新的双 Token，更新本地存储
    - 自动重试之前失败的 API 请求（用户完全无感知）
    - 收到 403 状态码：清除本地所有 Token，跳转到登录页，要求用户重新登录

### 阶段 4：用户主动登出

1. **客户端**：调用`/api/logout`接口，携带 Refresh Token
2. **服务器**：从数据库 / Redis 中删除该 Refresh Token
3. **客户端**：清除本地所有 Token，跳转到登录页

### 阶段 5：强制登出（服务器端主动吊销）

以下场景服务器需主动吊销用户的 Refresh Token：

- 用户修改密码
- 用户在其他设备登录（可选择 "踢出其他设备"）
- 账号被封禁
- 检测到异常登录（异地登录、异常设备）

服务器只需从数据库 / Redis 中删除对应的 Refresh Token 即可，此时即使攻击者窃取了旧的 Refresh Token，也无法再刷新获取新的 Access Token。

## 三、关键实现细节

### 1. Token 的生成方式

- **Access Token**：几乎都用**JWT（JSON Web Token）** 实现
    
    - 优点：无状态，服务器无需存储，验证速度快；可携带用户 ID、权限等自定义信息
    - 缺点：无法即时吊销，因此必须设置极短的有效期
    
- **Refresh Token**：**绝对不能用 JWT**，必须用**随机字符串**实现
    
    - 原因：Refresh Token 需要支持即时吊销，且必须存储在服务器端，用随机字符串更安全、更易管理
    - 生成方式：使用加密安全的随机数生成器（如 Node.js 的`crypto.randomBytes(32)`），长度至少 32 字节

### 2. 客户端存储方案（安全核心）

这是双 Token 模式最容易出错的地方，直接决定了系统的安全性：

| 存储位置            | 适用 Token      | 优点                             | 缺点                      | 安全建议                                                  |
| --------------- | ------------- | ------------------------------ | ----------------------- | ----------------------------------------------------- |
| 内存（Vuex/Redux）  | Access Token  | 完全免疫 XSS 攻击                    | 页面刷新后丢失                 | ✅ 首选方案，刷新页面时用 Refresh Token 重新获取 Access Token         |
| HttpOnly Cookie | Refresh Token | JavaScript 无法读取，免疫 XSS 攻击；自动携带 | 存在 CSRF 风险              | ✅ 强制使用，同时设置`Secure`（仅 HTTPS 传输）和`SameSite=Strict/Lax` |
| localStorage    | 不推荐任何 Token   | 持久化存储                          | 完全暴露给 XSS 攻击，一旦被窃取可长期使用 | ❌ 绝对禁止存储 Refresh Token，尽量不要存储 Access Token            |
| sessionStorage  | 不推荐任何 Token   | 关闭标签页后自动清除                     | 仍暴露给 XSS 攻击             | ❌ 不推荐                                                 |

### 3. 服务器端存储方案

- Refresh Token 必须存储在**Redis**中（推荐）或数据库中
- 存储结构示例（Redis Hash）：
    
```plaintext
    key: refresh_token:{token_value}
    value: {
      user_id: 123,
      device_id: "abc123", // 绑定设备，防止跨设备使用
      expires_at: 1718438400, // 过期时间戳
      created_at: 1717833600
    }
```
    
- 同时为每个用户维护一个 Refresh Token 列表，方便实现 "踢出所有设备" 功能：
    
```plaintext
    key: user_refresh_tokens:{user_id}
    value: ["token1", "token2", "token3"]
```

### 4. 过期时间设置建议

- **Access Token**：15 分钟～2 小时
    
    - 安全要求高的系统（金融、支付）：15 分钟
    - 普通业务系统：30 分钟～1 小时
    - 绝对不要超过 2 小时
    
- **Refresh Token**：7 天～30 天
    
    - 移动端 APP：30 天（用户登录频率低）
    - Web/H5：7 天（安全风险更高）
    - 可根据业务需求调整，但不要超过 90 天

### 5. 刷新接口的设计要点

- 刷新接口**只能接受 Refresh Token**，不能接受 Access Token
- 每次刷新必须生成**新的 Refresh Token**，旧的立即失效（防止 Refresh Token 被窃取后被多次使用）
- 对刷新接口进行**频率限制**（比如每分钟最多 5 次），防止暴力破解
- 刷新时验证设备信息，如果设备不匹配，拒绝刷新并要求用户重新登录

## 四、安全加固最佳实践

1. **全程 HTTPS**：所有接口必须使用 HTTPS 传输，防止中间人攻击窃取 Token
2. **Refresh Token 轮换**：每次刷新都生成新的 Refresh Token，旧的立即删除
3. **设备绑定**：将 Refresh Token 与设备指纹（User-Agent+IP + 设备唯一标识）绑定，异常设备无法使用
4. **黑名单机制**：对于已吊销但未过期的 Access Token，可存入 Redis 黑名单（过期时间与 Access Token 一致）
5. **异常检测**：检测到异地登录、多次登录失败、异常 API 调用时，强制吊销所有 Refresh Token
6. **登出彻底性**：用户登出时，必须同时删除服务器端和客户端的所有 Token
7. **前端防护**：做好 XSS 防护（输入过滤、输出转义、CSP 策略），防止 Access Token 被窃取

## 五、常见误区与问题解答

### 1. 为什么 Refresh Token 不能用来直接访问资源？

因为 Refresh Token 有效期很长，如果允许它直接访问资源，一旦泄露会造成长期安全风险。双 Token 的核心就是让高权限、长有效期的 Refresh Token 尽可能少地在网络中传输，只在 Access Token 过期时使用一次。

### 2. Refresh Token 过期了怎么办？

Refresh Token 过期意味着用户的 "长期登录状态" 失效，此时必须要求用户重新输入账号密码登录，这是安全与体验的平衡点。

### 3. 双 Token 能完全防止 Token 泄露吗？

不能。任何 Token 模式都无法完全防止 Token 泄露，但双 Token 模式能**最大限度降低泄露后的危害**：

- Access Token 有效期短，泄露后最多只能使用 15 分钟
- Refresh Token 存储在 HttpOnly Cookie 中，几乎不会被 XSS 攻击窃取
- 一旦发现泄露，服务器可立即吊销 Refresh Token，阻止攻击者继续获取新的 Access Token

### 4. 为什么 JWT 不能用于 Refresh Token？

JWT 是无状态的，服务器无法即时吊销。如果用 JWT 作为 Refresh Token，一旦泄露，攻击者可以在整个有效期内（比如 30 天）不断刷新获取新的 Access Token，服务器完全无法阻止。

**无状态（Stateless）的核心是：服务器不存储任何关于客户端会话的信息，所有必要的信息都由客户端自己携带**。

#### 1. 先理解 "有状态"：传统 Session 登录

传统的 Session 登录是典型的**有状态**认证：

1. 用户登录成功后，服务器生成一个`sessionId`
2. 服务器把`sessionId`和对应的用户信息（`{userId: 123, username: 'test'}`）存在自己的内存或数据库里
3. 服务器把`sessionId`返回给客户端，客户端存在 Cookie 里
4. 后续每次请求，客户端携带`sessionId`
5. 服务器收到请求后，**必须去数据库里查**这个`sessionId`对应的用户信息，才能知道是谁在访问

**有状态的本质**：服务器记住了每个客户端的状态，客户端只需要带一个 ID。

#### 2. 再理解 "无状态"：JWT 登录

JWT 登录是典型的**无状态**认证：

1. 用户登录成功后，服务器生成一个 JWT
2. JWT 本身就包含了所有必要的用户信息（`{userId: 123, username: 'test', exp: 1718438400}`）
3. 服务器用自己的私钥对 JWT 进行签名，然后返回给客户端
4. 后续每次请求，客户端携带完整的 JWT
5. 服务器收到请求后，**不需要去数据库查任何东西**，只需要用公钥验证 JWT 的签名是否正确，再看一下过期时间，就可以信任里面的用户信息

**无状态的本质**：服务器什么都不记，所有状态都由客户端自己携带，服务器只负责验证。

#### 3. 无状态的优缺点

- **优点**：
    
    - 性能极高：验证不需要查数据库，减少 IO 操作
    - 易于扩展：多服务器部署时，不需要做 Session 共享
    - 跨平台友好：不依赖 Cookie，适合 APP、小程序等非浏览器环境
    
- **缺点**：
    - **无法即时吊销**：这是最致命的缺点，也是我们需要双 Token 的根本原因
    - Token 体积较大：因为携带了用户信息，会增加请求头的大小

### 5. 页面刷新后 Access Token 丢失怎么办？

这是正常现象。页面刷新时，内存中的 Access Token 会被清除，此时客户端应立即调用刷新接口，用 Refresh Token 获取新的 Access Token，整个过程用户无感知。

## 六、极简代码示例（Node.js + Express）

### 后端核心代码

```javascript
const express = require('express');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const redis = require('redis');
const app = express();
const client = redis.createClient();

// 密钥
const ACCESS_TOKEN_SECRET = 'your-access-token-secret';
const REFRESH_TOKEN_EXPIRES_IN = 7 * 24 * 60 * 60; // 7天
const ACCESS_TOKEN_EXPIRES_IN = 15 * 60; // 15分钟

// 登录接口
app.post('/api/login', async (req, res) => {
  const { username, password } = req.body;
  
  // 验证账号密码（此处省略）
  const user = { id: 123, username: 'test' };
  
  // 生成Access Token
  const accessToken = jwt.sign(
    { userId: user.id, username: user.username },
    ACCESS_TOKEN_SECRET,
    { expiresIn: ACCESS_TOKEN_EXPIRES_IN }
  );
  
  // 生成Refresh Token
  const refreshToken = crypto.randomBytes(32).toString('hex');
  const refreshTokenKey = `refresh_token:${refreshToken}`;
  
  // 存储Refresh Token到Redis
  await client.hSet(refreshTokenKey, 'userId', user.id);
  await client.expire(refreshTokenKey, REFRESH_TOKEN_EXPIRES_IN);
  await client.sAdd(`user_refresh_tokens:${user.id}`, refreshToken);
  
  // 设置Refresh Token到HttpOnly Cookie
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: REFRESH_TOKEN_EXPIRES_IN * 1000
  });
  
  res.json({ accessToken });
});

// 刷新Token接口
app.post('/api/refresh-token', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) return res.sendStatus(403);
  
  const refreshTokenKey = `refresh_token:${refreshToken}`;
  const userId = await client.hGet(refreshTokenKey, 'userId');
  if (!userId) return res.sendStatus(403);
  
  // 删除旧的Refresh Token
  await client.del(refreshTokenKey);
  await client.sRem(`user_refresh_tokens:${userId}`, refreshToken);
  
  // 生成新的双Token
  const newAccessToken = jwt.sign(
    { userId: parseInt(userId) },
    ACCESS_TOKEN_SECRET,
    { expiresIn: ACCESS_TOKEN_EXPIRES_IN }
  );
  const newRefreshToken = crypto.randomBytes(32).toString('hex');
  const newRefreshTokenKey = `refresh_token:${newRefreshToken}`;
  
  await client.hSet(newRefreshTokenKey, 'userId', userId);
  await client.expire(newRefreshTokenKey, REFRESH_TOKEN_EXPIRES_IN);
  await client.sAdd(`user_refresh_tokens:${userId}`, newRefreshToken);
  
  res.cookie('refreshToken', newRefreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: REFRESH_TOKEN_EXPIRES_IN * 1000
  });
  
  res.json({ accessToken: newAccessToken });
});

// 登出接口
app.post('/api/logout', async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) return res.sendStatus(200);
  
  const refreshTokenKey = `refresh_token:${refreshToken}`;
  const userId = await client.hGet(refreshTokenKey, 'userId');
  
  if (userId) {
    await client.del(refreshTokenKey);
    await client.sRem(`user_refresh_tokens:${userId}`, refreshToken);
  }
  
  res.clearCookie('refreshToken');
  res.sendStatus(200);
});

// 受保护资源接口
app.get('/api/protected', (req, res) => {
  const authHeader = req.headers.authorization;
  const token = authHeader && authHeader.split(' ')[1];
  if (!token) return res.sendStatus(401);
  
  try {
    const user = jwt.verify(token, ACCESS_TOKEN_SECRET);
    res.json({ message: '受保护的资源数据', user });
  } catch (err) {
    res.sendStatus(401);
  }
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

### 前端核心逻辑（Axios 拦截器）

```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: 'http://localhost:3000/api',
  withCredentials: true // 允许跨域携带Cookie
});

// 请求拦截器：自动携带Access Token
api.interceptors.request.use(
  (config) => {
    const accessToken = localStorage.getItem('accessToken'); // 实际推荐存在内存中
    if (accessToken) {
      config.headers.Authorization = `Bearer ${accessToken}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 响应拦截器：自动处理Token过期
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) prom.reject(error);
    else prom.resolve(token);
  });
  failedQueue = [];
};

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    
    // 如果是401且不是刷新请求
    if (error.response.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // 如果正在刷新，将请求加入队列
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then(token => {
            originalRequest.headers.Authorization = `Bearer ${token}`;
            return api(originalRequest);
          })
          .catch(err => Promise.reject(err));
      }
      
      originalRequest._retry = true;
      isRefreshing = true;
      
      try {
        // 调用刷新接口
        const { data } = await api.post('/refresh-token');
        const newAccessToken = data.accessToken;
        
        // 更新本地Access Token
        localStorage.setItem('accessToken', newAccessToken);
        
        // 处理队列中的请求
        processQueue(null, newAccessToken);
        
        // 重试当前请求
        originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
        return api(originalRequest);
      } catch (refreshError) {
        // 刷新失败，跳转到登录页
        processQueue(refreshError, null);
        localStorage.removeItem('accessToken');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }
    
    return Promise.reject(error);
  }
);

export default api;
```

## 七、总结

双 Token 登录模式的核心是**职责分离与风险隔离**：

- 用短有效期的 Access Token 承担高频访问，降低泄露后的危害
- 用高安全存储的 Refresh Token 解决续期问题，保证用户体验
- 所有安全措施都围绕保护 Refresh Token 展开，因为它是整个认证体系的核心

这是目前工业界最成熟、最安全的无状态身份认证方案，广泛应用于 Web、H5、小程序、APP 等所有端的登录系统。