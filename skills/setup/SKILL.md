---
name: glab:setup
description: Diagnose glab CLI installation and authentication state for the GitLab instance, then output a step-by-step Chinese setup guide for the user to follow manually. Use when glab is missing or unauthenticated, when 'glab auth status' fails, when other glab:* skills hit missing dependency, or when user says 装 glab / 配置 GitLab CLI / install glab / setup glab.
allowed-tools: Bash
---

# /glab:setup — glab 状态诊断 + 教程输出

## 行为约束（必读）

**本 skill 是只读诊断 + 教程输出，绝对不**：

- ❌ 不执行 `winget install` / `scoop install` / `choco install`
- ❌ 不执行 `glab auth login`
- ❌ 不写入或修改任何 token / 配置文件

**只允许做**：

- ✅ 跑 `glab --version`、`glab auth status` 等只读检测命令
- ✅ 把对应教程**作为响应文本**输出给用户

让用户自己运行命令、自己粘贴 token，Claude 不代劳。

---

## 第一步：检测 glab 状态

按顺序跑（捕获 stderr，不要因失败中断）：

```bash
glab --version 2>&1 || echo "__GLAB_MISSING__"
```

```bash
glab auth status --hostname git.snowsse.cn 2>&1 || true
```

根据输出归类到 4 种状态之一：

| 状态 | 判定 |
|---|---|
| **A. 未安装** | 第一条命令输出包含 `__GLAB_MISSING__` 或 `command not found` |
| **B. 已装未认证** | 第一条 OK；第二条输出含 `No token found` 或 `not logged in` |
| **C. token 失效** | 第一条 OK；第二条输出含 `401` 或 `Unauthorized` |
| **D. 已就绪** | 两条都 OK，第二条出现 `Logged in to git.snowsse.cn as <username>` |

如果用户在 PowerShell 里 `glab` 找不到但环境变量里其实有，可能是当前会话 PATH 未刷新。在 PowerShell 下可以追加跑：

```powershell
$env:Path = [Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [Environment]::GetEnvironmentVariable("Path","User"); glab --version
```

如果刷新后能识别，归为 **B / C / D** 之一并在教程里提醒用户"重启终端"。

---

## 第二步：根据状态输出教程

### A. glab 未安装

输出以下内容（原样发给用户）：

```
❌ 检测到本机未安装 glab CLI。

请在 PowerShell 任选一种方式安装（推荐 winget）：

【方式 1：winget（Windows 自带，推荐）】
  winget install -e --id GLab.GLab --accept-source-agreements --accept-package-agreements

【方式 2：Scoop】
  scoop install glab

【方式 3：Chocolatey（需管理员）】
  choco install glab

安装后请：
1. 关闭当前终端，重新开一个 PowerShell（让 PATH 生效）
2. 跑 glab --version 验证
3. 回到 Claude Code 运行 /glab:setup 继续认证

完成后请重新运行你的原请求。
```

### B. 已装未认证

输出：

```
✓ glab 已安装（版本 <从第一条输出中提取>）
❌ 但尚未认证到 git.snowsse.cn。

【步骤 1：创建 Personal Access Token】

1. 浏览器打开（任选可用的）：
   - https://git.snowsse.cn/-/user_settings/personal_access_tokens   （GitLab 16+）
   - https://git.snowsse.cn/-/profile/personal_access_tokens          （旧版）

2. 滚动到 "Personal Access Tokens" 区域，填表：
   - Token name：glab-cli（任意，方便识别）
   - Expiration：留空或填 1 年后
   - Scopes 必须勾选：
     ☑ api
     ☑ read_user
     ☑ read_repository
     ☑ write_repository

3. 点 "Create personal access token"

4. 页面顶部会出现 glpat-xxxxxxxx... 字符串，立刻点 📋 复制按钮（关闭页面就再也看不到）

【步骤 2：在 PowerShell 注入 token】

  glab auth login --hostname git.snowsse.cn --token <粘贴 token>

【步骤 3：验证】

  glab auth status

看到 "✓ Logged in to git.snowsse.cn as <你的用户名>" 即成功。

完成后请重新运行你的原请求。
```

### C. token 失效（401）

输出：

```
⚠ glab 已安装且配置了 token，但 GitLab 返回 401 Unauthorized。

可能原因（按概率排）：
1. Token 复制时丢字符（GitLab 16+ token 是 26 字符，老版是 20 字符）
2. Token 已被 revoke 或过期
3. 创建时漏勾 api scope
4. 公司 GitLab 启用了 PAT 审批策略，新建 token 默认 Pending（需管理员批准）

建议直接重做 token，按以下步骤：

1. 浏览器打开 https://git.snowsse.cn/-/user_settings/personal_access_tokens
2. 滚动到 "Active personal access tokens" 列表，确认旧 token 是 Active 还是 Pending
   - 若是 Pending，找管理员审批，或新建一个再观察
3. 创建新 token：
   - Scopes 务必勾 api / read_user / read_repository / write_repository
4. 复制新 token 后，在 PowerShell 跑：

  glab auth login --hostname git.snowsse.cn --token <新 token>

5. 验证：glab auth status

完成后请重新运行你的原请求。
```

### D. 已就绪

输出：

```
✓ glab 已安装并认证为 <username>

可以使用以下 skill：
  /glab:mr              查看当前分支 MR
  /glab:mr-create       创建新 MR
  /glab:mr-diff         MR 变更中文总结
  /glab:pipeline        当前分支 pipeline 状态
  /glab:commits-ahead   本分支领先 commits
  /glab:review-mr       多维 review
  /glab:review-fixup    处理 AI review 评论 → 修复 plan
```

---

## 第三步：终止

输出教程后**不要**继续做其他事。等用户根据教程操作完成后，主动运行原请求即可。
