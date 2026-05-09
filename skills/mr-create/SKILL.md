---
name: glab:mr-create
description: Create a new GitLab merge request from the current branch. Generates Chinese title and description from git commit messages. Use when user says 创建 MR / 提个 PR / 提 MR / open a merge request / new MR.
allowed-tools: Bash
---

# /glab:mr-create — 从当前分支创建 MR

## Preconditions

依赖 `glab` CLI 已安装并完成 git.snowsse.cn 认证。

执行前先跑：

```bash
glab auth status
```

如果失败：调用 `glab:setup` skill 输出教程给用户，提示完成后重跑，**终止本 skill**。

---

## 参数

- 目标分支（target branch）：自然语言里抽取，否则**默认 `develop`**
  - 用户说 "创建 MR 到 main" → target = `main`
  - 用户说 "提 MR" → target = `develop`

---

## 主流程

### 1. 检查前置条件

```bash
git rev-parse --abbrev-ref HEAD              # 当前分支
git status --porcelain                        # 看是否有未提交的修改
```

- 如果 HEAD 是 `main` / `develop` / `master`，告诉用户"不应该从主干分支创建 MR"，结束
- 如果有未提交修改，提示用户"先 `git commit` 再创建 MR（避免 MR 跟实际分支不一致）"，让用户选择是否继续

### 2. 检查是否已有 MR

```bash
glab mr list --source-branch <当前分支> --output json
```

如果非空，提示"当前分支已有 MR !<iid>，无需重复创建。可用 `/glab:mr` 查看"，结束。

### 3. 拿 commits 和 diff stat

```bash
git log <target>..HEAD --oneline --no-merges
git diff <target>..HEAD --stat
```

如果 `git log` 返回空，说明本分支跟 target 没差异，告诉用户"没有变更可提交"，结束。

### 4. Claude 生成中文标题和描述

根据上一步的 commits + diff stat，按以下规则生成：

**标题**（≤ 70 字符）：
- 主导意图：从 commit message 里提炼，例如 "feat(logger): 接入 SnowLogger 并补充 desensitization"
- 多个 commit 时取最重要的主题
- 跟 commit message 风格保持一致（中文 + conventional 前缀）

**描述**（Markdown）：

```markdown
## 变更概要

<2-3 句话总结此次 MR 做了什么、为什么>

## 主要改动

- <模块 1>: 具体做了什么
- <模块 2>: 具体做了什么

## 影响范围

<列出受影响的文件类别：源代码 / 配置 / 测试 / 文档>

## 测试方式

- [ ] 本地 typecheck 通过
- [ ] 单元测试通过
- [ ] 手工验证（说明操作步骤）

## Commit 列表

<git log --oneline 的内容>
```

### 5. 给用户预览，确认后再执行

把生成的标题和描述完整展示给用户，问："以上标题和描述是否 OK？OK 我就执行 `glab mr create`，否则你可以让我调整。"

得到肯定后再执行。

### 6. 执行创建（默认 draft）

```bash
glab mr create \
  --target-branch <target> \
  --title "<标题>" \
  --description "<描述>" \
  --draft
```

- 默认创建为 **Draft**，安全
- 如果用户明确说"直接 ready 不要 draft"，则去掉 `--draft`

### 7. 输出

执行成功后展示：

- MR 编号 + URL
- 提示："已创建为 Draft。代码就绪后跑 `glab mr update <iid> --ready` 取消 draft；或者 `/glab:mr` 查看详情"
