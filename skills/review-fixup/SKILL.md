---
name: glab:review-fixup
description: Pull review comments left by the snow_dev_ai bot on a GitLab MR, evaluate each finding's validity using project conventions from llmdoc/, and produce a prioritized fix plan grouped by severity. Use when user says 处理 AI review / AI review 评论 / snow_dev_ai 反馈 / fix plan for AI review / 看看 AI review 说了啥.
allowed-tools: Bash, Read, Glob, Grep
---

# /glab:review-fixup — AI Review 评论解析与修复 Plan

## Preconditions

依赖 `glab` CLI 已安装并完成 git.snowsse.cn 认证。还需要 `jq`（可与 git for windows 一起装；通常 `glab` 装好后系统会有）。

执行前先跑：

```bash
glab auth status && jq --version
```

如果 `glab` 失败：调用 `glab:setup` 输出教程，**终止本 skill**。
如果 `jq` 失败：告诉用户"`jq` 未安装，请 `winget install -e --id jqlang.jq`，安装后重试"，**终止本 skill**。

---

## 参数

- MR 编号（iid）：**必填**。如果用户没传：
  - 提示："/glab:review-fixup 需要 MR 编号，例如 `/glab:review-fixup 10`"
  - 终止

---

## 主流程

### 1. 拉数据

#### 1a. 拿项目路径并 URL 编码

```bash
PROJECT_PATH=$(glab repo view --output json | jq -r '.path_with_namespace')
PROJECT_PATH_ENCODED=$(echo -n "$PROJECT_PATH" | jq -sRr @uri)
```

例：`snow_workspace_group/sws-browser-extension` → `snow_workspace_group%2Fsws-browser-extension`

#### 1b. 拉所有 discussions（带行内 position）

```bash
glab api "projects/${PROJECT_PATH_ENCODED}/merge_requests/<MR>/discussions" \
  --paginate \
  > /tmp/glab-mr-<MR>-discussions.json
```

注意：`glab api --paginate` 会自动翻页直到拿到全部数据。

#### 1c. 拉 diff

```bash
glab mr diff <MR> > /tmp/glab-mr-<MR>-diff.patch
```

### 2. 过滤 snow_dev_ai 评论

用 jq 从 discussions 提取 AI 留下的 notes：

```bash
jq '
  [
    .[] |
    .notes[] |
    select(.author.username == "snow_dev_ai") |
    {
      discussion_id: .discussion_id,
      note_id: .id,
      body: .body,
      file: (.position.new_path // .position.old_path // null),
      new_line: .position.new_line,
      old_line: .position.old_line,
      resolved: .resolvable and .resolved,
      created_at: .created_at
    }
  ]
' /tmp/glab-mr-<MR>-discussions.json > /tmp/glab-mr-<MR>-ai-findings.json
```

读这个文件，得到 finding 数组。

#### 边界情况

- **数组为空**：明确输出 "MR !<iid> 上未发现 `snow_dev_ai` 的 review 记录。可能 AI 还没跑完，或没找到需要建议的代码"，结束
- **全部 resolved**：提醒"所有 AI 评论已被标记 resolved，无需处理。仍要看的话告诉我"
- **只有部分 resolved**：默认只处理未 resolved 的；如果用户明确要看全部，再处理

### 3. 加载 llmdoc 上下文（动态、可选）

检测 cwd 下 `llmdoc/index.md`：

#### 若存在

**a. 读索引**：`Read llmdoc/index.md`

**b. 必读规范**：Glob `llmdoc/must/*.md`，全部 Read

**c. 编码规范**：Read `llmdoc/reference/coding-conventions.md`（若存在）

**d. 按 finding 涉及关键词定向加载**：

聚合所有 finding 涉及的文件路径和评论 body 关键词，匹配 index.md 描述：

| 关键词信号 | 加载 |
|---|---|
| logger / log / SnowLogger | `llmdoc/reference/snow-logger-guide.md` + `llmdoc/guides/how-to-add-snow-logger.md` |
| 脱敏 / desensitization / mask | `llmdoc/architecture/data-desensitization-architecture.md` + `llmdoc/guides/how-to-add-desensitization-rule.md` |
| recording / 录屏 / rrweb | `llmdoc/architecture/recording-architecture.md` + `llmdoc/guides/how-to-debug-recording.md` |
| service worker / background / KeepServiceAlive | `llmdoc/reference/chrome-extension-mv3-guide.md` + 涉及 worker 的 architecture 文档 |
| proxy / PAC | `llmdoc/architecture/snow-proxy-architecture.md` + `llmdoc/reference/snow-proxy-manifest-reference.md` |
| shared / KeepServiceAlive / indexedDB | `llmdoc/architecture/shared-library-architecture.md` + `llmdoc/reference/shared-api-reference.md` + `llmdoc/reference/indexeddb-state-manager-guide.md` |
| CI / GitLab CI / 飞书 | `llmdoc/reference/gitlab-ci-feishu-notification.md` |
| DOM / 性能 / layout | `llmdoc/reference/chrome-dom-performance.md` |

#### 若不存在

跳过；evaluation 仅基于代码本身，并在报告头部标注 "未加载 llmdoc 项目规范"。

### 4. 逐条评估合理性

对每个 finding，按以下五分类标签：

| 分类 | 含义 | 行动 |
|---|---|---|
| **valid bug** 🔴 | 真问题（逻辑错、安全漏洞、明确违反规范） | 必须修，给 diff |
| **valid improvement** 🟡 | 真合理建议（命名、可读性、性能微优化） | 建议修，给 diff |
| **style preference** ⚪ | 风格分歧（如分号、换行、命名风格） | 看团队约定，默认不修 |
| **false positive** ⚪ | AI 误报（基于上下文不该报） | 不修，但要写明不修原因 |
| **uncertain** 🔵 | 缺信息无法判断（如缺业务背景） | 留给用户决策 |

#### 评估方式

对每个 finding：

1. **读对应文件的实际上下文**：用 Read 工具读 finding.file，重点看 finding.new_line 附近 ±20 行
2. **读相关的 llmdoc 文档**（如已加载）寻找规范依据
3. **判断**：AI 提议是否合理？是否在当前上下文成立？
4. **打标签**：从上面 5 类里选一个
5. **写依据**：1-2 句话说明为什么这么判

### 5. 输出修复 plan

格式：

```markdown
# AI Review Fix Plan: MR !<iid>

> 项目: <path_with_namespace>
> 评估时间: <now>
> 项目规范来源: <llmdoc/ 已加载 X 篇 | 未加载>

## 评估摘要

- AI Findings 总数: N
- 🔴 必须修复（valid bug）: A
- 🟡 建议改进（valid improvement）: B
- ⚪ 可忽略（style/false-positive）: C
- 🔵 待决策（uncertain）: D

---

## 🔴 必须修复

### 1. `<file>:<line>` — <一句话标题>

**AI 评论**:
> <原文，可能很长，可摘要核心>

**评估**: 同意。<理由>

**依据**: <llmdoc 路径或通用最佳实践>

**修复**:
\`\`\`diff
- <原代码>
+ <改后代码>
\`\`\`

### 2. ...

---

## 🟡 建议改进

<同样格式>

---

## ⚪ 可忽略项

| 文件:行 | AI 提议 | 不修原因 |
|---|---|---|
| `foo.ts:42` | 建议把 const 改 let | false positive — 这里就是不需要变量重赋值 |
| `bar.ts:88` | 建议加注释 | style — 函数名已经够清晰 |

---

## 🔵 待决策

### 1. `<file>:<line>` — <标题>

**AI 评论**: ...
**为什么不能立刻判断**: 缺业务上下文 / 缺设计意图 / 涉及历史遗留
**需要你确认**: <具体的问题>

---

## 推荐执行顺序

1. 先修 🔴 N 个问题，跑 typecheck / 单测验证
2. 再处理 🟡 建议（视时间）
3. 提交建议格式：`fix(review): apply AI review feedback for MR !<iid>`
4. 跑 `pnpm typecheck && pnpm test`（项目用 pnpm）/ `npm test`（如用 npm）/ 或本仓库实际测试命令
5. push 后 GitLab AI 会自动重新 review；若仍有评论再跑 `/glab:review-fixup <iid>`

---

## 在 GitLab 上的后续操作（可选，由你决定）

修复完成并 push 后，可以手动在 MR 页面把已处理的 AI 评论标记 resolved。本 skill **不会自动**调用 `glab` resolve API。
```

### 6. 关键约束

- ✅ 只产出修复 plan，**绝不直接改代码**
- ✅ 所有 `glab api` 调用必须 `--paginate`，避免漏分页
- ✅ URL 编码用 `jq @uri`，绝不用 `sed` 手 hack
- ✅ 临时文件统一存 `/tmp/glab-mr-<iid>-*`，便于 debug 时排查
- ✅ llmdoc 缺失时优雅降级，不报错
- ❌ 不自动 `glab discussion resolve`（让用户决定）
- ❌ 不自动 push 修复（仅 plan）
- ❌ 不自动给 MR 加评论
