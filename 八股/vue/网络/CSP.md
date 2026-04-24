内容安全策略（Content Security Policy，简称 CSP）是一种由浏览器实施的强大安全机制。你可以把它理解为网站给浏览器下达的一道**“白名单”指令**，明确规定了页面可以加载哪些资源（如脚本、样式、图片、字体等），以及从何处加载。

它的核心思想是**“默认拒绝，授权放行”**。任何未在白名单中明确允许的资源，浏览器都会拒绝加载或执行。这种机制从根本上改变了传统 Web 安全“默认允许所有”的模式，为防御跨站脚本攻击（XSS）、数据注入和点击劫持等常见威胁提供了强有力的保障。

### 🛡️ 为什么需要 CSP？

要理解 CSP 的价值，首先要明白传统安全机制的局限。

- **同源策略 (SOP) 的边界**：同源策略是浏览器的基础安全模型，它像一个“围墙”，阻止 A 网站的脚本读取 B 网站的数据。但它无法防止“围墙内部”的破坏。
- **XSS 攻击的本质**：如果黑客通过漏洞（如未过滤的用户输入）在你的网页中注入了一段恶意脚本，浏览器会认为这段脚本是“自己人”（因为它来自当前网站的源），并赋予它完整的权限去窃取用户 Cookie、监听用户输入等。

**CSP 的作用就是在这道“围墙”内部再设置一道“安检”**。即使黑客成功注入了恶意脚本，只要这个脚本的来源不在 CSP 的白名单内，或者它是以不被允许的方式（如内联脚本）执行的，浏览器就会直接将其拦截，让攻击失效。

### ⚙️ 如何部署 CSP？

CSP 策略主要通过两种方式传递给浏览器，推荐优先使用第一种。

1. **HTTP 响应头 (推荐)**  
    这是最安全、最可靠的方式。服务器在返回 HTML 页面时，通过 `Content-Security-Policy` 响应头携带策略。
    
```http
    Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com;
```
    
   这种方式适用于整个网站，并且支持所有 CSP 指令。
    
2. **HTML `<meta>` 标签**  
    在无法修改服务器配置的场景下（如静态页面托管），可以在 HTML 的 `<head>` 标签内通过 `<meta>` 标签来声明策略。
    
```html
    <meta http-equiv="Content-Security-Policy" content="default-src 'self'">
```
    
   这种方式仅对当前页面生效，且功能受限（例如不支持 `report-uri` 指令）。


### 📜 核心指令与关键字

CSP 策略由一系列指令（Directive）和源值（Source Value）构成。

#### 常用指令 (Directive)

每个指令负责管控一类资源的加载。

|指令|作用|示例|
|---|---|---|
|`default-src`|**全局默认策略**。为所有未单独指定的资源类型提供兜底规则。|`'self'`|
|`script-src`|管控 JavaScript 的加载和执行，是防御 XSS 的核心。|`'self' https://cdn.example.com`|
|`style-src`|管控 CSS 样式表的加载。|`'self' 'unsafe-inline'`|
|`img-src`|管控图片资源的加载。|`'self' data: https:`|
|`connect-src`|管控 AJAX、Fetch、WebSocket 等网络请求的目标地址。|`'self' https://api.example.com`|
|`frame-src`|管控 `<iframe>` 可以嵌入的页面来源，用于防止点击劫持。|`'none'`|

#### 常用源值 (Source Value)

源值用于定义白名单的具体内容。

|源值|含义|注意事项|
|---|---|---|
|`'self'`|仅允许来自**当前网站同源**（协议、域名、端口均一致）的资源。|最基础、最安全的配置。|
|`'none'`|**完全禁止**加载该类资源。|常用于 `object-src` 以禁用 Flash 等插件。|
|`'unsafe-inline'`|允许执行**内联**的脚本或样式（如 `<script>...</script>` 或 `onclick="..."`）。|**高危**，会极大削弱 CSP 对 XSS 的防御能力，应尽量避免使用。|
|`'unsafe-eval'`|允许使用 `eval()`、`new Function()` 等从字符串动态执行代码。|**高危**，同样是 XSS 攻击的常见入口，应避免使用。|
|`https://example.com`|允许来自**指定域名**的资源。|可用于信任的第三方 CDN。|

### 📝 策略示例与最佳实践

一个典型的 CSP 策略可能如下所示：

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; object-src 'none';
```

这条策略的含义是：

- 所有资源默认只允许从本站加载 (`default-src 'self'`)。
- 脚本可以从本站和 `cdn.jsdelivr.net` 加载。
- 样式可以从本站加载，也允许内联样式。
- 图片可以从本站、Data URI (`data:`) 和任何 HTTPS 网站加载。
- 禁止加载任何 `<object>` 插件资源。

#### 最佳实践：从报告模式开始

直接部署一个严格的 CSP 策略可能会误伤正常的业务代码，导致网站功能异常。因此，最佳实践是先在**报告模式 (Report-Only Mode)** 下进行测试。

1. **使用报告模式**：将 HTTP 响应头设置为 `Content-Security-Policy-Report-Only`。
2. **配置报告地址**：通过 `report-uri` 或 `report-to` 指令指定一个接收违规报告的服务器地址。
3. **收集与分析**：在此模式下，浏览器不会拦截任何违规资源，但会将所有违反策略的行为以 JSON 格式发送到你的报告地址。
4. **迭代与强制**：分析这些报告，修复或调整你的策略，直到确认没有误报后，再将响应头切换回强制执行的 `Content-Security-Policy`。