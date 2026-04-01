这两个属性是前端开发中保护敏感 Cookie（如登录态 Session ID、用户 Token）的核心手段，直接关系到 XSS 攻击防护和数据传输安全，重点讲你在前端开发中能感知到的部分。

## HttpOnly 属性

### 核心作用

**禁止前端 JavaScript 通过 `document.cookie` 读取或修改该 Cookie**，是缓解 XSS（跨站脚本攻击）偷 Cookie 的最直接方式。

### 前端能看到什么

1. **浏览器 DevTools 里的表现**：
    
    打开 DevTools → Application → Cookies，能看到该 Cookie 有一个 `HttpOnly` 的勾选标记（已勾选），说明属性生效。
    
2. **JS 里的表现**：
    
    即使 Cookie 存在，执行 `console.log(document.cookie)` 也**完全看不到**这个 Cookie 的值，更无法通过 JS 修改它。
### 实际开发中的影响

- **正常情况**：前端不需要读敏感 Cookie（如 Session ID），浏览器会自动在请求里带，设 `HttpOnly` 不影响正常接口调用。
- **防 XSS 偷 Cookie**：如果页面不幸有 XSS 漏洞，攻击者注入的 JS 脚本拿不到这个 Cookie，就没法冒充用户登录（会话劫持）。

### 怎么设置（前端要知道格式）

虽然是后端在响应头里设，但你要认识这个格式：

```http
Set-Cookie: sessionId=abc123xyz; HttpOnly; Path=/
```

### 前端常见坑

- **“为什么我拿不到登录态？”**：如果登录态 Cookie 设了 `HttpOnly`，`document.cookie` 拿不到是**正常的**，别慌，接口请求会自动带。
- **不是防 XSS 万能**：它只防 “偷 Cookie”，XSS 还能伪造用户点击、偷页面其他数据，所以该做的输入过滤、输出编码还是要做。

## Secure 属性

### 核心作用

**确保 Cookie 只在 HTTPS 加密请求中传输**，HTTP 请求里绝对不会带，防止网络传输中被中间人截获。

### 前端能看到什么

1. **DevTools 里的表现**：
    
    - Application → Cookies 里能看到 `Secure` 勾选标记；
    - Network 面板看请求：如果是 HTTP 请求，Request Headers 里的 Cookie 字段**没有这个 Cookie**；如果是 HTTPS 请求，才会自动带上。
    
2. **本地开发的例外**：
    
    浏览器对 `localhost`/`127.0.0.1` 有特殊照顾 —— 即使是 HTTP，也可能携带 `Secure` Cookie（方便本地开发），但**生产环境必须用 HTTPS**，否则这个 Cookie 发不出去。

### 实际开发中的影响

- **生产环境必须 HTTPS**：如果你的网站只有 HTTP，设了 `Secure` 的 Cookie 会 “消失”—— 浏览器不会带它发请求，导致登录态失效。
- **混合内容要注意**：如果页面是 HTTPS，但里面嵌了 HTTP 的图片 / 接口，那个 HTTP 接口请求里不会带 `Secure` Cookie。

### 怎么设置（前端要知道格式）

同样是后端在响应头里设：

```http
Set-Cookie: sessionId=abc123xyz; Secure; Path=/
```

### 前端常见坑

- **“本地开发好好的，上线 Cookie 没了？”**：检查上线环境是不是 HTTPS，如果是 HTTP，把 `Secure` 去掉（或者赶紧配 HTTPS）。

## 前端开发最佳实践：组合使用

对于登录态、用户凭证这类敏感 Cookie，**必须同时加 HttpOnly + Secure，再配 SameSite**，形成三层防护：

```http
Set-Cookie: sessionId=abc123xyz; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400
```

- `HttpOnly`：JS 读不到，防 XSS 偷；
- `Secure`：只走 HTTPS，防传输截获；
- `SameSite=Lax`：防 CSRF（跨站请求伪造），别让别的网站随便用你的 Cookie 发请求。

## 前端要记住的 3 个关键点

1. **HttpOnly 是 “防 JS 读”**：设了就别想用 `document.cookie` 拿登录态，这是安全的；
2. **Secure 是 “只走 HTTPS”**：生产环境必须配 HTTPS，否则 Cookie 发不出去；
3. **敏感 Cookie 两个都要加**：登录态、Token 这类，别省这两个属性。