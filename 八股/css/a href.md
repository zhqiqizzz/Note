`<a>` 是 HTML 中用于创建**超链接**的核心标签，而 `href`（Hypertext Reference）是它最重要的属性，用于指定链接的目标地址。通过 `<a href>`，用户可以点击跳转到其他页面、文件、锚点位置，甚至触发邮件 / 电话等系统功能。

## 一、基础语法

```html
<a href="目标地址">链接文本/内容</a>
```

- **`href`**：必填属性，指定链接的目标。
- **链接内容**：可以是文本、图片、图标等，用户点击该内容即可跳转。

## 二、常见的 `href` 值类型

### 1. 普通 URL（绝对路径）

跳转到其他网站的完整地址，需包含协议（`http://`/`https://`）：

```html
<!-- 跳转到 Vue 官方文档 -->
<a href="https://vuejs.org">Vue 官方文档</a>

<!-- 跳转到百度 -->
<a href="https://www.baidu.com">百度一下</a>
```

### 2. 相对路径

跳转到同一网站内的其他页面或文件，无需协议和域名：

```html
<!-- 跳转到当前目录下的 about.html -->
<a href="about.html">关于我们</a>

<!-- 跳转到上级目录下的 contact.html -->
<a href="../contact.html">联系我们</a>

<!-- 跳转到当前目录下的 docs 文件夹中的 index.html -->
<a href="./docs/index.html">文档首页</a>

<!-- 跳转到根目录下的 login.html -->
<a href="/login.html">登录</a>
```

### 3. 锚点链接（页面内 / 页面间跳转）

跳转到页面的**指定位置**，通过 `#` 加目标元素的 `id` 实现：

```html
<!-- 1. 页面内跳转：点击后滚动到当前页面 id="section1" 的元素 -->
<a href="#section1">跳转到第一部分</a>

<!-- 目标元素 -->
<div id="section1">第一部分内容</div>

<!-- 2. 页面间跳转：跳转到 about.html 页面的 id="team" 位置 -->
<a href="about.html#team">关于我们 - 团队介绍</a>
```

### 4. 邮件链接（`mailto:`）

点击后自动打开系统默认的邮件客户端，并预填收件人：

```html
<!-- 预填收件人 -->
<a href="mailto:contact@example.com">联系我们</a>

<!-- 预填收件人、主题、正文 -->
<a href="mailto:contact@example.com?subject=咨询&body=您好，我想咨询...">发送咨询邮件</a>
```

### 5. 电话链接（`tel:`）

移动端点击后自动打开拨号界面，并预填电话号码：

```html
<a href="tel:13800138000">拨打客服电话</a>
```

### 6. 文件下载链接

如果 `href` 指向的是浏览器无法直接打开的文件（如 `.zip`、`.pdf`、`.exe`），浏览器会自动下载该文件：

```html
<!-- 下载 PDF 文件 -->
<a href="/files/manual.pdf">下载使用手册</a>

<!-- 下载压缩包 -->
<a href="/files/project.zip">下载项目文件</a>
```

_提示：也可以配合 `download` 属性强制下载，即使文件能被浏览器打开（如图片）：_

```html
<!-- 强制下载图片，而不是在浏览器中打开 -->
<a href="/images/logo.png" download="logo.png">下载 Logo</a>
```

### 7. 空链接 / 占位链接

#### （1）`href="#"`

点击后跳转到页面顶部，但会**改变 URL 哈希**，且会触发页面滚动：

```html
<a href="#">回到顶部（不推荐，会改变 URL）</a>
```

#### （2）`href="javascript:void(0)"`

点击后无任何操作，不会改变 URL 也不会滚动，但**不推荐**（不符合语义化，且可能有安全风险）：

```html
<a href="javascript:void(0)">无操作链接（不推荐）</a>
```

#### ✅ 推荐替代方案

如果只是需要一个可点击的元素但不需要跳转，**用 `<button>` 标签**（更符合语义化）：

```html
<button @click="handleClick">点击执行操作</button>
```

如果必须用 `<a>`，可以通过 JS 阻止默认行为（Vue 中用 `@click.prevent`）：

```html
<a href="#" @click.prevent="handleClick">点击执行操作</a>
```

## 三、配合 `href` 常用的属性

### 1. `target`：控制链接的打开方式

|属性值|说明|
|---|---|
|`_self`（默认）|在当前窗口 / 标签页打开，替换当前页面。|
|`_blank`|在**新窗口 / 新标签页**打开。|
|`_parent`|在父框架中打开（很少用，用于框架集页面）。|
|`_top`|在顶层窗口中打开（很少用，用于框架集页面）。|

**示例**：

```html
<!-- 在新标签页打开 Vue 官网 -->
<a href="https://vuejs.org" target="_blank">Vue 官方文档（新标签页）</a>
```

#### ⚠️ 安全提示：`target="_blank"` 需配合 `rel="noopener noreferrer"`

当使用 `target="_blank"` 时，新打开的页面可以通过 `window.opener` 访问原页面的 `window` 对象，存在安全风险。需添加 `rel="noopener noreferrer"` 禁止这种访问：

```html
<!-- 安全的新标签页打开方式 -->
<a href="https://vuejs.org" target="_blank" rel="noopener noreferrer">
  Vue 官方文档（新标签页）
</a>
```

### 2. `rel`：定义链接与当前页面的关系

除了上面的 `noopener noreferrer`，还有几个常用值：

|属性值|说明|
|---|---|
|`nofollow`|告诉搜索引擎不要追踪该链接（不传递权重），用于广告、第三方链接等。|
|`external`|表示该链接指向外部网站。|

**示例**：

```html
<a href="https://example.com" rel="nofollow external">第三方链接</a>
```

## 四、最佳实践与注意事项

1. **语义化使用**：
    
    - 如果是**导航 / 跳转**，用 `<a href>`。
    - 如果是**执行操作**（如提交表单、打开弹窗），用 `<button>`。
    
2. **避免空链接**：不要用 `href="#"` 或 `href="javascript:void(0)"` 做无意义跳转，优先用 `<button>`。
    
3. **新标签页打开的安全**：`target="_blank"` 必须加 `rel="noopener noreferrer"`。
    
4. **链接文本要有意义**：不要用 “点击这里”“详情” 等模糊文本，要用描述性文字（如 “查看 Vue 官方文档”），提升可访问性和 SEO。
    
5. **相对路径 vs 绝对路径**：
    
    - 同一网站内的跳转，用**相对路径**（更灵活，迁移域名时无需修改）。
    - 跳转到其他网站，用**绝对路径**（必须包含协议）。