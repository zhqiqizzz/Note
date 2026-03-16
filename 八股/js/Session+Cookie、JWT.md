这两种是目前前端最常用的身份认证方案，核心区别在于**登录态的存储位置**和**验证方式**，下面从原理、流程、前端操作、安全等维度逐一拆解，只讲你开发中会接触到的部分。

## 一、Session+Cookie 认证方案

### 核心原理

**“后端存数据，前端存钥匙”**：

- 后端：生成并存储完整的用户会话数据（Session），比如用户 ID、登录时间、权限等，存储位置通常是内存、Redis 或数据库；
- 前端：只存一个**Session ID**（通过 Cookie），作为 “钥匙”，后续请求时浏览器自动带上，后端用这个 ID 去查对应的 Session 数据，确认用户身份。

### 完整流程（前端视角）

1. **登录阶段**
    
    前端发送账号密码 → 后端验证通过 → 后端生成 Session 并存储 → 后端通过 `Set-Cookie` 响应头把 Session ID 种到浏览器 Cookie 里。

```http
    // 后端响应头示例（关键）
    Set-Cookie: sessionId=abc123xyz; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400
```
    
2. **后续请求阶段**
    
    前端发任何接口请求 → **浏览器自动把 Cookie 带在请求头里** → 后端拿到 `sessionId` → 去存储里查对应的 Session → 确认身份后返回数据。

```http
    // 前端请求头（浏览器自动加，不用你写）
    Cookie: sessionId=abc123xyz
```

### 前端具体操作

Session+Cookie 对前端非常 “省心”，大部分工作浏览器和后端做了：

- **登录时**：不用管 Cookie，只要发账号密码请求，后端 `Set-Cookie` 后浏览器会自动存；
- **发请求时**：不用手动加任何头，浏览器会自动带 Cookie（前提是接口和页面同域，或跨域时后端配了 `Credentials: include`）；
- **查看 Cookie**：打开 DevTools → Application → Cookies，能看到 `sessionId`，如果有 `HttpOnly` 标记，JS 读不到（正常的）。

⚠️ **跨域注意**：如果是前后端分离跨域（比如前端 `localhost:3000`，后端 `api.example.com`），需要：

1. 前端请求时加 `credentials: 'include'`（fetch）或 `withCredentials: true`（Axios）；
2. 后端响应头加 `Access-Control-Allow-Credentials: true`，且 `Access-Control-Allow-Origin` 不能是 `*`，必须是具体域名。

### 安全要点（前端要知道后端怎么设）

敏感 Cookie（如 `sessionId`）必须同时加这三个属性，后端在 `Set-Cookie` 里配：

1. **HttpOnly**：禁止 JS 通过 `document.cookie` 读，防 XSS 偷；
2. **Secure**：只在 HTTPS 请求里带，防传输截获；
3. **SameSite=Lax/Strict**：防 CSRF（跨站请求伪造），别让别的网站随便用你的 Cookie 发请求。

## 二、JWT 认证方案

### 核心原理

**“无状态，自包含”**：

- 后端：不存任何会话数据，只生成一个**自包含的 JWT 字符串**（里面直接存用户 ID、过期时间等），用密钥签名后返回给前端；
- 前端：存完整的 JWT，后续请求时手动（或自动）带给后端，后端通过**验证签名**确认 JWT 没被篡改，再读里面的用户信息确认身份。

### JWT 结构回顾（前端要能看懂）

JWT 是三个用 `.` 分隔的 Base64 编码字符串：

```plaintext
Header.Payload.Signature
```

- **Header**：存加密算法（如 `HS256`）和类型（固定 `JWT`），Base64 编码；
- **Payload**：存实际数据（如 `sub` 用户 ID、`exp` 过期时间），**Base64 编码但不加密**（别放敏感信息）；
- **Signature**：后端用密钥对 `Header.Payload` 加密生成，防篡改。

### 完整流程（前端视角）

1. **登录阶段**
    
    前端发送账号密码 → 后端验证通过 → 后端生成 JWT → 放在**响应体**里返回给前端。
    
```javascript
    // 前端登录请求示例（Axios）
    async function login(username, password) {
      const res = await axios.post('/api/login', { username, password });
      const token = res.data.token; // 后端返回 JWT，比如 "eyJhbGci...xxx"
      // 接下来：把 token 存起来
    }
```
    
2. **后续请求阶段**
    
    前端发接口请求 → **手动把 JWT 放在请求头的 `Authorization` 字段里**（格式固定 `Bearer <token>`） → 后端验证签名和过期时间 → 确认身份后返回数据。
    
```javascript
    // 前端普通请求示例（Axios）
    async function getUserInfo() {
      const token = localStorage.getItem('token'); // 从存储里拿
      const res = await axios.get('/api/user', {
        headers: {
          'Authorization': `Bearer ${token}` // 关键：Bearer 开头加空格
        }
      });
      return res.data;
    }
```

### 前端具体操作

JWT 对前端的 “操作量” 比 Session+Cookie 多一点，核心是**存储和携带**：

#### 1. 存储 JWT：三种方式选一个

|存储方式|优点|缺点|推荐场景|
|---|---|---|---|
|`localStorage`|永久存储（除非手动删），容量大（5MB）|易受 XSS 攻击，攻击者偷到 token 就能冒充用户|非敏感 token，或配合严格 XSS 防护|
|`sessionStorage`|关闭浏览器就清空，更安全|易受 XSS 攻击，刷新页面还在但关闭就没|临时登录态（如 “记住我” 关闭时）|
|**HttpOnly Cookie**|防 XSS 偷（JS 读不到），浏览器自动带|可能受 CSRF 攻击，需配合 `SameSite` 属性|**最推荐**，敏感登录态优先用这个|

⚠️ **最佳实践**：如果后端支持，让后端把 JWT 放在 `HttpOnly; Secure; SameSite=Lax` 的 Cookie 里，前端不用手动存和带，和 Session+Cookie 一样省心，还能防 XSS。

#### 2. 解码 JWT 看过期（前端常用）

JWT 的 Payload 里有 `exp`（过期时间，Unix 时间戳），前端可以写个工具函数解码，提前判断是否过期：

```javascript
// 解码 JWT Payload 的工具函数
function parseJwtPayload(token) {
  try {
    const base64Url = token.split('.')[1]; // 取中间的 Payload
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
    const jsonPayload = decodeURIComponent(atob(base64).split('').map(c => 
      '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2)
    ).join(''));
    return JSON.parse(jsonPayload);
  } catch (e) {
    return null;
  }
}

// 检查是否过期
function isTokenExpired(token) {
  const payload = parseJwtPayload(token);
  if (!payload) return true;
  const now = Date.now() / 1000; // 当前时间转 Unix 时间戳
  return payload.exp < now;
}
```

#### 3. 双 Token 机制（优化体验）

为了避免 JWT 过期后用户频繁重新登录，常用 “双 Token 机制”：

- **Access Token**：短时效（1-2 小时），用于普通接口请求；
- **Refresh Token**：长时效（7-30 天），用于刷新 Access Token。

流程：Access Token 过期 → 前端用 Refresh Token 调用 `/api/refresh` 接口 → 后端返回新的 Access Token → 前端继续用新 Token 发请求，不用重新登录。

### 安全要点

1. **别在 Payload 放敏感信息**：Base64 能解码，密码、身份证号绝对不能放；
2. **优先存 HttpOnly Cookie**：防 XSS 偷 token，比 `localStorage` 安全；
3. **必须用 HTTPS**：防止 JWT 在传输中被中间人截获；
4. **过期时间别太长**：Access Token 设 1-2 小时，降低泄露风险。

## 三、详细对比（前端视角表格）

|对比项|Session+Cookie|JWT|
|---|---|---|
|**登录态存储位置**|后端存 Session，前端只存 Session ID|前端存完整 JWT，后端不存|
|**携带方式**|浏览器自动带 Cookie|手动放 `Authorization: Bearer <token>` 头，或自动带 Cookie|
|**前端操作量**|少（浏览器自动处理）|稍多（需存储、手动加头、处理过期）|
|**扩展性**|差（后端集群需共享 Session，如 Redis）|好（无状态，后端集群不用共享）|
|**XSS 防护**|靠 `HttpOnly` 防偷|存 `localStorage` 易被偷，存 `HttpOnly` 可防|
|**CSRF 防护**|靠 `SameSite` 防|放 `Authorization` 头天然防 CSRF|
|**适用场景**|传统单体项目、对 CSRF 防护要求高|前后端分离、后端集群、移动端 APP（不方便带 Cookie）|

## 四、前端选型建议

- **优先选 Session+Cookie**：如果是传统项目、后端是单体、主要在浏览器端用，Session+Cookie 更省心，安全也容易做；
- **选 JWT**：如果是前后端分离跨域、后端是集群、需要支持移动端 APP（如 React Native、Flutter，不方便自动带 Cookie），JWT 更灵活。

不管选哪种，**敏感登录态都优先用 HttpOnly+Secure+SameSite Cookie 存**，这是目前最安全的方式。