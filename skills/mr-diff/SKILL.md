---
name: glab:mr-diff
description: Show GitLab MR diff and summarize changes in Chinese (grouped by file category, with risk highlights). Use when user says 看 MR 改了什么 / summarize MR diff / 这个 PR 改了哪些 / MR 变更总结.
allowed-tools: Bash, Read
---

# /glab:mr-diff — MR 变更中文总结

## Preconditions

依赖 `glab` CLI 已安装并完成 git.snowsse.cn 认证。

执行前先跑：`glab auth status`。失败则调用 `glab:setup` 输出教程，**终止本 skill**。

---

## 参数

- MR 编号（iid）：
  - 用户给了 → 直接用
  - 没给 → 拿当前分支查找：`git rev-parse --abbrev-ref HEAD` + `glab mr list --source-branch <分支> --output json`
    - 找到 1 条 → 用它的 iid
    - 0 条或多条 → 让用户明确指定 MR 编号

---

## 主流程

### 1. 拉 diff 到临时文件（避免上下文爆炸）

```bash
glab mr diff <iid> > /tmp/glab-mr-<iid>-diff.patch 2>/dev/null
```

如果非空：用 Read 工具读取这个文件。

### 2. 拉 MR 元数据（可选，用于上下文）

```bash
glab mr view <iid>
```

提取标题、源/目标分支、commit 数。

### 3. 按文件类别分组

把改动文件按以下 5 类分组：

| 类别 | 匹配规则（优先级从上到下） |
|---|---|
| 测试 | 路径含 `__tests__` / `.test.` / `.spec.` / `tests/` |
| 配置 | `*.json` / `*.yaml` / `*.yml` / `*.toml` / `vite.config.*` / `tsconfig*` / `package.json` |
| 文档 | `*.md` / `docs/` / `llmdoc/` / `README*` |
| 样式 | `*.css` / `*.scss` / `*.less` |
| 源代码 | 其他（`*.ts` / `*.js` / `*.tsx` / `*.vue` 等） |

### 4. 中文总结输出

按下面格式：

```
## MR !<iid>: <title>

- 源分支: <source> → <target>
- Commit 数: <N>
- 改动文件: <total> 个（+<additions> / -<deletions> 行）

## 变更分组

### 源代码（X 个文件）
- `<file>`：一句话说明改了什么意图（不是机械列举改了哪几行）
- ...

### 配置（Y 个文件）
- ...

### 测试（Z 个文件）
- ...

### 文档（W 个文件）
- ...

## 风险点

<3-5 条 Claude 的判断；如果没有明显风险，写"未发现明显风险"。常见风险类型：
  - 删除/重命名公共 API 但未提供向后兼容
  - 修改了核心 service worker / background 逻辑（针对浏览器扩展项目）
  - 改动了认证 / 权限 / 加密相关代码
  - 删除了测试且未补充新测试
  - 改了构建配置可能影响产物大小
>

## 建议

- <可选：建议哪些地方再 review 一遍>
- <可选：是否补测试 / 文档>
```

### 5. 控制输出长度

如果 diff 超过 1000 行，每个分组里"重要改动"列前 5 个文件 + 一条 "...等共 N 个文件"。让用户能扫读。

总结篇幅控制在 500 字以内（不算分组列表）。
