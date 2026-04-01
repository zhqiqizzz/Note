打开你的目标目录`D:\BaiduNetdiskDownload\Note\Vue`，在空白处按住 Shift + 右键，打开 PowerShell / 终端，依次执行以下命令，彻底清理错误配置：

```bash
# 如果有 查看并删除旧的错误origin远程地址
git remote -v
git remote remove origin

# 1. 初始化git仓库
git init
# 可选择配置.gitignore
.idea
.obsidian
.trash
.git
# 2. 重新绑定你的GitHub远程仓库
git remote add origin https://github.com/Github用户名/文件.git

# 3. 确保本地分支名为main，和远程仓库对齐
git branch -M main

# 4. 初始化用户信息（首次使用Git必须配置）
git config --global user.name "你的GitHub用户名"
git config --global user.email "你的GitHub绑定邮箱"
```

测试提交

```bash
# 查看文件状态，确认README.md被Git识别
git status

# 添加所有文件到暂存区
git add .

# 提交到本地仓库
git commit -m "init: 初始化Obsidian笔记库"

# 推送到GitHub远程仓库，完成首次绑定
git push -u origin main
```

如果你在新设备上已经建好了一个空文件夹（比如也叫 `Note`），想直接把 GitHub 上的东西拉到这个空文件夹里：
### 操作步骤：

```bash
#1. 在新设备上打开 PowerShell，**进入你的现有空文件夹**：
# 假设你的现有文件夹在这，根据实际情况改路径
cd D:\你的\现有\文件夹路径\Note

# 2. 初始化 Git（让这个空文件夹变成 Git 仓库）：
git init

# 3. 绑定你的 GitHub 远程仓库：
git remote add origin https://github.com/GitHub用户名/文件夹.git

# 4. 拉取 GitHub 上的内容：
git pull origin main

# 4. 确保本地分支名为main，和远程仓库对齐
git branch -M main

# 4. 初始化用户信息（首次使用Git必须配置）
git config --global user.name "你的GitHub用户名"
git config --global user.email "你的GitHub绑定邮箱"
```

### SSH

HTTPS 协议持续无法连接时，直接切换为 SSH 协议访问仓库，彻底避开 443 端口连接问题：

1. 本地生成 SSH 密钥对（替换为你的 GitHub 注册邮箱）

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

全程回车，无需设置密码，完成密钥生成。

2. 复制公钥内容
    
    - Windows：`C:\Users\你的用户名\.ssh\id_ed25519.pub`
    - Mac/Linux：`~/.ssh/id_ed25519.pub`
        
        用文本编辑器打开文件，复制里面的全部内容。

2. 绑定到 GitHub 账号

打开 GitHub → 进入 Settings → SSH and GPG keys → New SSH key，将复制的公钥粘贴进去，保存。

2. 验证 SSH 连通性

```bash
ssh -T git@github.com
```

提示 `Hi 你的用户名! You've successfully authenticated...` 即为连接成功。

5. 切换仓库远程地址为 SSH 地址

```bash
git remote set-url origin git@github.com:zhqiqizzz/Note.git
```

之后即可正常执行 pull/push 等所有 Git 操作。

**若密钥不匹配**
#### 解决方法：清除旧的 GitHub 主机密钥记录

只需一条命令即可清除本地缓存的旧密钥，解决此问题：
##### 1. 执行清除命令

在当前终端（PowerShell 或 CMD）中直接执行：

```bash
ssh-keygen -R github.com
```

_(注意：`-R` 是大写的)_

这条命令会自动在你的 `known_hosts` 文件中删除所有与 `github.com` 相关的旧密钥记录。

##### 2. 重新连接并验证

清除后，再次执行连接命令：

```bash
ssh -T git@github.com
```

此时系统会提示你是否接受新的主机密钥，输入 `yes` 并回车即可：

```plaintext
The authenticity of host 'github.com (xx.xx.xx.xx)' can't be established.
ED25519 key fingerprint is SHA256:xxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

如果看到 `Hi 用户名! You've successfully authenticated...` 的提示，说明连接成功，问题已解决。