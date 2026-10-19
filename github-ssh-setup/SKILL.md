---
name: github-ssh-setup
description: Windows 环境下供 Agent 自主规划与执行的单账号 GitHub SSH 密钥配置工作流。当用户要求配置、生成、绑定或修复 Windows 下的 GitHub SSH 密钥、设置 Git SSH 免密连接或排查 SSH 连接超时/权限报错时触发。
---
# github-ssh-setup

供 Agent 在 Windows 环境（Git Bash 或 PowerShell）下自主规划并静默执行单账号 GitHub SSH 密钥检查、无交互生成、配置写入、公钥上传及连接验证的标准化任务流。

## When to Use

  - 用户要求在 Windows 上生成、配置或连接 GitHub SSH 密钥
  - 用户要求将本地环境或 Git 仓库配置为 GitHub SSH 免密访问
  - 用户要求修复 `Permission denied (publickey)` 或 `port 22: Connection timed out` 等 GitHub SSH 连接失败问题

## Steps

Agent 收到任务后应自主按以下阶段推进并直接执行命令，优先通过静默参数完成自动化操作，避免因等待交互输入导致进程挂起。

### Step 1: 自主探测环境与现有密钥

1.  检查 `$HOME/.ssh`（PowerShell 为 `$env:USERPROFILE\.ssh`）目录下是否已存在 `id_ed25519` 与 `id_ed25519.pub`。
2.  若已存在密钥对，直接复用并跳至 Step 3，切勿覆盖已有密钥。
3.  若不存在密钥，默认采用邮箱 `goojoer@gmail.com` 作为注释标识（也可参考 `git config --global user.email` 获取）。

### Step 2: 静默无交互生成 Ed25519 密钥

在确保 `.ssh` 目录存在后，必须携带 `-f` 与 `-N ""` 参数执行无交互生成，防止 `ssh-keygen` 阻塞等待标准输入：

  - **Git Bash 执行**：

``` bash
mkdir -p ~/.ssh
ssh-keygen -t ed25519 -C "goojoer@gmail.com" -f ~/.ssh/id_ed25519 -N "" -q
```

  - **PowerShell 执行**：

``` powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.ssh" | Out-Null
ssh-keygen -t ed25519 -C "goojoer@gmail.com" -f "$env:USERPROFILE\.ssh\id_ed25519" -N '""' -q
```

  - **验证标准**：确认 `id_ed25519`（私钥）和 `id_ed25519.pub`（公钥）两个文件均已生成且非空。

### Step 3: 写入 SSH Config 并加载密钥

Windows 下不同终端（PowerShell、CMD、Git Bash）的 `ssh-agent` 状态常不互通。Agent 应直接将密钥路径写入 `~/.ssh/config`，确保任何终端调用 Git 时都能自动匹配密钥，无需依赖后台代理服务常驻：

1.  检查 `~/.ssh/config`，若尚未包含 `Host github.com` 配置块，则追加以下内容：

``` text
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

2.  若当前环境启用了 `ssh-agent`，同步执行 `ssh-add ~/.ssh/id_ed25519` 加载私钥。

### Step 4: 自动注册或导出公钥到 GitHub

按以下优先级将 `~/.ssh/id_ed25519.pub` 注册到用户的 GitHub 账号：

1.  **优先尝试 GitHub CLI 自动上传**：检测是否存在已登录的 `gh` 命令（`gh auth status`）。若已就绪，直接执行：

``` bash
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Windows-$(hostname)"
```

2.  **剪贴板自动复制与降级处理**：若未安装或未登录 `gh`，自动将公钥写入系统剪贴板（Git Bash 执行 `clip < ~/.ssh/id_ed25519.pub`，PowerShell 执行 `Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" | Set-Clipboard`），并在输出中展示公钥全文与 `https://github.com/settings/keys` 链接（或调用浏览器工具前往该页面），提示用户粘贴保存。

### Step 5: 非交互式连接验证与 443 端口自动回退

公钥添加完成后，执行非交互式 SSH 握手测试。必须附加 `-o StrictHostKeyChecking=accept-new` 以自动接受首次连接的 GitHub 主机指纹，防止进程挂起等待输入 `yes`：

``` bash
ssh -T -o StrictHostKeyChecking=accept-new -o ConnectTimeout=10 git@github.com
```

  - **结果判定**：GitHub 认证成功时进程退出码为 `1`（因为不提供 shell 访问），只要输出包含 `successfully authenticated` 即视为验证通过。
  - **22 端口超时自动修复**：若返回 `Connection timed out` 或 `Connection reset by peer`，Agent 应立即自主修改 `~/.ssh/config`，切换为 443 端口并重新执行上面的测试命令：

``` text
Host github.com
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

### Step 6: 自动切换当前仓库远程地址（若适用）

若当前工作目录处于 Git 仓库内，执行 `git remote get-url origin` 检查远程地址。若地址格式为 `https://github.com/<owner>/<repo>.git`，自动执行以下命令切换为 SSH 协议：

``` bash
git remote set-url origin git@github.com:<owner>/<repo>.git
```

## Gotchas

  - **交互式挂起陷阱**：Agent 执行 `ssh-keygen` 时若漏写 `-f` 或 `-N ""`，或者执行首次 `ssh -T` 时漏写 `-o StrictHostKeyChecking=accept-new`，会导致命令无限期挂起等待终端输入。
  - **退出码误判**：`ssh -T git@github.com` 在认证成功时固定返回退出码 `1`（而非 `0`），必须根据标准错误输出中的 `successfully authenticated` 文本判断成败，切勿仅凭非零退出码判定失败。
  - **Windows PowerShell 空密码转义**：在 PowerShell 中向原生程序传递空字符串参数时，有时 `-N ""` 会被剥离导致报错 `too many arguments` 或再次提示输入密码。若在 PowerShell 中遇到该问题，可改用 `cmd /c 'ssh-keygen -t ed25519 -C "comment" -f "%USERPROFILE%\.ssh\id_ed25519" -N "" -q'` 确保空密码参数正确传递。
  - **严禁泄露私钥**：读取或展示密钥时只能操作 `.pub` 后缀文件，绝不能在日志或回复中打印 `id_ed25519` 私钥内容。
