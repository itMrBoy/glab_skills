---
name: glab:mr
description: View the GitLab merge request associated with the current git branch. Use when user says 看当前 MR / 当前分支 MR / view current MR / show me the merge request / 这个分支有 MR 吗.
allowed-tools: Bash
---

# /glab:mr — 查看当前分支的 MR

## Preconditions

依赖 `glab` CLI 已安装并完成 git.snowsse.cn 认证。

执行前先跑：

```bash
glab auth status
```

如果失败（未安装 / 未认证 / token 过期 / command not found）：

1. 调用 `glab:setup` skill 拿到诊断结果与教程
2. 把教程**完整原样**输出给用户
3. 在末尾追加："完成上述步骤后请重新运行 /glab:mr"
4. **终止本 skill 执行**（不要继续）

---

## 主流程

### 1. 拿当前分支

```bash
git rev-parse --abbrev-ref HEAD
```

如果在 detached HEAD（输出是 `HEAD`），告诉用户"当前不在分支上，无法定位 MR"，结束。

### 2. 列出当前分支对应的 MR

```bash
glab mr list --source-branch <分支名> --output json
```

解析 JSON 数组：

- **空数组**：输出 "当前分支 `<分支>` 没有对应的 open MR。可用 `/glab:mr-create` 创建。"，结束
- **1 条**：取它的 `iid`
- **多条**：列出所有 `iid` + `title` + `state`，让用户指定要看哪个

### 3. 查看 MR 详情

```bash
glab mr view <iid>
```

### 4. 中文总结输出

格式：

```
## MR !<iid>: <title>

- 状态: <draft/open/merged/closed>
- 分支: <source_branch> → <target_branch>
- 作者: <author>
- 创建于: <created_at>

### 流水线
<pipeline 状态：success / failed / running / skipped；如失败列出哪些 job>

### 讨论
- 总评论数: N
- 未解决讨论: M（如有）
- 最近 3 条评论作者: a, b, c

### 链接
<web_url>
```

如果 `glab mr view` 输出里包含 "approval rules" / "approvals required"，也一并提及（"审批: 已 X/Y"）。

### 5. 给操作建议

最后用一行提示：

- 如果状态是 draft：建议 `/glab:mr-diff <iid>` 先看 diff，再 `glab mr update <iid> --ready` 标记 ready
- 如果有未解决讨论：建议 `/glab:review-fixup <iid>` 处理 AI review 评论
- 如果 pipeline 失败：建议 `/glab:pipeline` 看详情
