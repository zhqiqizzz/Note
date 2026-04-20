第三方登录的核心是基于 **OAuth 2.0 授权框架**（或其身份层扩展 **OpenID Connect**），让用户无需在你的网站注册账号，直接通过微信、GitHub、Google 等第三方平台的身份授权，安全、快速地完成登录。

它的本质是：**用户在第三方平台确认授权后，第三方平台给你的网站颁发一个「临时访问凭证」，你的网站用这个凭证换取用户的基本身份信息，最终完成账号关联与登录**。

## 一、核心前置知识：OAuth 2.0 授权码流程

目前所有主流第三方登录，都使用 **OAuth 2.0 授权码流程（Authorization Code Flow）**，这是最安全、最标准的流程，全程在服务器端完成，不暴露敏感凭证。

### 完整流程的 7 个核心步骤

| 步骤  | 动作                      | 参与方            | 核心说明                                                                                    |
| --- | ----------------------- | -------------- | --------------------------------------------------------------------------------------- |
| 1   | 构造授权链接，引导用户跳转           | 你的网站前端 → 第三方平台 | 你的网站生成带 `Client ID`、`回调地址`、`state`（防 CSRF）的授权链接，用户点击跳转到第三方平台的授权页面                       |
| 2   | 用户在第三方平台确认授权            | 用户 → 第三方平台     | 用户在第三方平台的页面上，确认同意授权你的网站获取其基本信息                                                          |
| 3   | 第三方平台回调你的网站，返回授权码       | 第三方平台 → 你的网站后端 | 用户授权后，第三方平台自动跳转到你预先配置的 `回调地址`，并在 URL 参数中带上 `code`（授权码）和 `state`                         |
| 4   | 你的后端验证 state，防止 CSRF    | 你的网站后端         | 后端验证回调的 `state` 是否和步骤 1 生成的一致，防止跨站请求伪造攻击                                                |
| 5   | 用授权码换取 Access Token     | 你的网站后端 → 第三方平台 | 后端用 `code` + `Client ID` + `Client Secret`，向第三方平台的 Token 接口发起请求，换取 `Access Token`（访问令牌） |
| 6   | 用 Access Token 获取用户基本信息 | 你的网站后端 → 第三方平台 | 后端用 `Access Token`，向第三方平台的用户信息接口发起请求，获取用户的唯一标识（openid/unionid）、昵称、头像等基本信息               |
| 7   | 关联本地账号，完成登录             | 你的网站后端 → 前端    | 后端用用户的唯一标识，在你的数据库中查找或创建对应的本地账号，生成你的网站的登录态（Session/Token），返回给前端，完成登录                     |

## 二、具体实现步骤（以 GitHub 第三方登录为例）

GitHub 是最适合个人开发者测试的第三方平台，无需企业资质，注册流程简单，接口文档清晰。我们以 **Node.js + Express** 为例，完整实现 GitHub 第三方登录。

### 第一步：在第三方平台注册应用，获取凭证

1. 登录 GitHub，进入 [Settings → Developer settings → OAuth Apps](https://github.com/settings/developers)
2. 点击「New OAuth App」，填写应用信息：
    
    - **Application name**：你的应用名称（用户授权时会看到）
    - **Homepage URL**：你的网站首页地址（本地测试用 `http://localhost:3000`）
    - **Authorization callback URL**：**回调地址**（必须和代码中一致，本地测试用 `http://localhost:3000/auth/github/callback`）
    
3. 注册成功后，获取两个核心凭证：
    
    - `Client ID`：公开标识，可放在前端
    - `Client Secret`：**核心密钥，必须保存在服务器端，绝对不能暴露在前端**
    

### 第二步：项目初始化与依赖安装

```bash
# 创建项目目录
mkdir github-oauth-demo
cd github-oauth-demo

# 初始化 package.json
npm init -y

# 安装依赖
npm install express axios express-session
```

### 第三步：完整代码实现

创建 `server.js` 文件，代码如下：

```javascript
const express = require('express');
const axios = require('axios');
const session = require('express-session');
const crypto = require('crypto');

const app = express();
const PORT = 3000;

// -------------------------- 配置信息（请替换为你自己的） --------------------------
const GITHUB_CLIENT_ID = '你的Client ID';
const GITHUB_CLIENT_SECRET = '你的Client Secret';
const GITHUB_REDIRECT_URI = 'http://localhost:3000/auth/github/callback';

// -------------------------- 中间件配置 --------------------------
// 配置 Session，用于存储 state 和用户登录态
app.use(session({
  secret: 'your-secret-key-keep-it-safe', // 替换为你的随机密钥
  resave: false,
  saveUninitialized: false,
  cookie: { secure: false, httpOnly: true } // 生产环境请开启 secure: true
}));

// -------------------------- 路由实现 --------------------------

// 1. 首页：显示登录按钮或用户信息
app.get('/', (req, res) => {
  if (req.session.user) {
    // 已登录，显示用户信息
    res.send(`
      <h1>登录成功！</h1>
      <p>昵称：${req.session.user.login}</p>
      <img src="${req.session.user.avatar_url}" width="100">
      <br><a href="/logout">退出登录</a>
    `);
  } else {
    // 未登录，显示 GitHub 登录按钮
    res.send('<h1>请登录</h1><a href="/auth/github">使用 GitHub 登录</a>');
  }
});

// 2. 第一步：构造授权链接，引导用户跳转到 GitHub
app.get('/auth/github', (req, res) => {
  // 生成随机 state，防止 CSRF 攻击
  const state = crypto.randomBytes(16).toString('hex');
  req.session.oauthState = state; // 存入 Session，后续验证用

  // 构造 GitHub 授权链接
  const authUrl = new URL('https://github.com/login/oauth/authorize');
  authUrl.searchParams.set('client_id', GITHUB_CLIENT_ID);
  authUrl.searchParams.set('redirect_uri', GITHUB_REDIRECT_URI);
  authUrl.searchParams.set('scope', 'user:email'); // 授权范围：获取用户基本信息和邮箱
  authUrl.searchParams.set('state', state);

  // 重定向到 GitHub 授权页面
  res.redirect(authUrl.toString());
});

// 3. 第三步到第七步：处理回调，换取 Token，获取用户信息，完成登录
app.get('/auth/github/callback', async (req, res) => {
  const { code, state } = req.query;

  // 验证 state，防止 CSRF 攻击
  if (!state || state !== req.session.oauthState) {
    return res.status(403).send('无效的 state 参数，可能是 CSRF 攻击');
  }
  delete req.session.oauthState; // 验证通过后删除，防止重复使用

  try {
    // -------------------------- 第五步：用 code 换取 Access Token --------------------------
    const tokenResponse = await axios.post(
      'https://github.com/login/oauth/access_token',
      {
        client_id: GITHUB_CLIENT_ID,
        client_secret: GITHUB_CLIENT_SECRET,
        code: code,
        redirect_uri: GITHUB_REDIRECT_URI
      },
      {
        headers: { Accept: 'application/json' }
      }
    );
    const accessToken = tokenResponse.data.access_token;

    // -------------------------- 第六步：用 Access Token 获取用户基本信息 --------------------------
    const userResponse = await axios.get('https://api.github.com/user', {
      headers: { Authorization: `Bearer ${accessToken}` }
    });
    const githubUser = userResponse.data;

    // -------------------------- 第七步：关联本地账号，完成登录 --------------------------
    // 这里的 githubUser.id 是用户在 GitHub 的唯一标识，用它来关联你的本地账号
    // 实际开发中，你需要查询数据库：
    // 1. 如果该 GitHub ID 已关联本地账号，直接登录
    // 2. 如果未关联，创建一个新的本地账号，并关联该 GitHub ID
    console.log('获取到的 GitHub 用户信息：', githubUser);

    // 简化示例：直接将用户信息存入 Session，完成登录
    req.session.user = githubUser;

    // 登录成功，跳转到首页
    res.redirect('/');
  } catch (error) {
    console.error('OAuth 流程出错：', error.response?.data || error.message);
    res.status(500).send('登录失败，请重试');
  }
});

// 退出登录
app.get('/logout', (req, res) => {
  req.session.destroy();
  res.redirect('/');
});

// 启动服务
app.listen(PORT, () => {
  console.log(`服务已启动：http://localhost:${PORT}`);
});
```

### 第四步：运行测试

1. 把代码中的 `GITHUB_CLIENT_ID` 和 `GITHUB_CLIENT_SECRET` 替换为你自己的
2. 运行 `node server.js`
3. 浏览器访问 `http://localhost:3000`，点击「使用 GitHub 登录」，即可测试完整流程

## 三、主流第三方平台对比

|第三方平台|适用场景|开发门槛|核心文档|
|---|---|---|---|
|GitHub|技术类网站、开发者工具|低，个人可直接注册|[GitHub OAuth 文档](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)|
|微信开放平台（App 登录）|移动 App、PC 网站扫码登录|中，需企业资质认证|[微信开放平台文档](https://developers.weixin.qq.com/doc/oplatform/Website_App/WeChat_Login/Wechat_Login.html)|
|微信公众平台（公众号授权）|微信内 H5 页面|中，需公众号|[微信公众平台文档](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/Wechat_webpage_authorization.html)|
|Google|海外用户、全球化网站|低，个人可直接注册|[Google Identity 文档](https://developers.google.com/identity/protocols/oauth2)|
|QQ / 微博|国内普通用户|中，部分需企业资质|QQ 互联 / 微博开放平台文档|

## 四、核心安全注意事项

1. **Client Secret 绝对保密**：必须保存在服务器端的环境变量或配置文件中，**绝对不能提交到代码仓库、绝对不能暴露在前端代码中**。
2. **必须验证 state 参数**：步骤 1 生成随机 state 存入 Session，步骤 4 回调时必须验证 state 一致，防止 CSRF 跨站请求伪造攻击。
3. **回调地址白名单**：必须在第三方平台严格配置回调地址白名单，只允许指定的回调地址接收授权码，防止授权码被窃取。
4. **授权码只能使用一次**：授权码 `code` 有效期极短（通常 5-10 分钟），且只能使用一次，使用后立即失效，防止重复使用。
5. **唯一标识关联账号**：必须用第三方平台返回的 `openid`/`unionid`/`id` 作为用户的唯一标识关联本地账号，**绝对不能用昵称、邮箱等可变更的信息**。
6. **HTTPS 强制加密**：生产环境必须使用 HTTPS，防止授权码、Access Token 在传输过程中被窃听。
7. **最小授权范围**：只请求必要的授权范围（`scope`），比如只需要用户昵称头像，就不要请求获取用户的好友列表、私信等敏感权限。

## 五、常见问题处理

### 1. 第三方登录后，如何让用户绑定手机号 / 完善信息？

- 用户第一次用第三方登录时，你的数据库创建一个新账号，但标记为「未完善信息」。
- 前端检测到该状态，弹出完善信息弹窗，引导用户绑定手机号、设置密码等，完成后再正常使用。

### 2. 如何实现「第三方登录 + 本地账号登录」的互通？

- 在用户信息完善页面，提供「绑定已有本地账号」的选项，让用户输入本地账号的用户名密码，验证通过后，将第三方唯一标识关联到该本地账号。
- 后续用户无论是用第三方登录还是本地账号登录，都能登录到同一个账号。

### 3. Access Token 过期了怎么办？

- 大部分第三方平台的 Access Token 有效期较短（几小时到几天），如果你的网站需要长期访问第三方平台的接口，需要在获取 Token 时同时获取 `Refresh Token`，用 Refresh Token 刷新 Access Token。
- 如果只是用于登录，不需要长期访问第三方接口，Access Token 过期不影响你的网站的登录态，因为你已经完成了账号关联，生成了自己的登录凭证。

## 六、总结

第三方登录的核心是 **OAuth 2.0 授权码流程**，关键在于：

1. 前端只负责引导用户跳转，所有敏感操作（换取 Token、获取用户信息）都在服务器端完成。
2. 严格验证 state，保护 Client Secret，配置回调白名单，确保安全。
3. 用第三方唯一标识关联本地账号，完成登录态的切换。