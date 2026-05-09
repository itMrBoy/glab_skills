# Review Pipeline — `review-mr` 与 `review-fixup` 双路

这是 `glab` plugin 相对原生 glab 的差异化价值：把 GitLab AI bot 的产出和项目专属 `llmdoc/` 知识结合，分两条独立路径输出"评审"或"修复 plan"。两条路是**独立兄弟**，**互不调用**，但共享多套基础设施。

## 双路对比一览

| 维度 | `review-mr` | `review-fixup` |
|---|---|---|
| 角色 | 主动评审任意 MR | 解析 GitLab `snow_dev_ai` bot 评论 → 评估合理性 → 修复 plan |
| 触发短语 | review MR / 评审 MR / 帮我看看这个 MR / code review | 处理 AI review / AI review 评论 / snow_dev_ai 反馈 / fix plan for AI review |
| MR iid | **必填**（无当前分支 fallback） | **必填**（无当前分支 fallback） |
| 多一道前置 | — | `jq --version` 检查 |
| 数据获取 | `glab mr view --comments` + `glab mr diff` | `glab repo view → jq @uri → glab api .../discussions --paginate` + `glab mr diff` |
| 是否需 jq | 否 | **是**（强依赖） |
| 主分类 | 6 维度（correctness/security/performance/readability/test/convention） | 5 桶（🔴 valid bug / 🟡 valid improvement / ⚪ style / ⚪ false positive / 🔵 uncertain） |
| 输出 | 评审报告 | 修复 plan（per-file/per-line + diff 块） |
| llmdoc 加载 | 4 步（index → must → topic-targeted → coding-conventions） | 4 步同 + 8 行 keyword→doc 路由表 |
| 安全边界 | 不动代码、不在 GitLab 加评论 | 不动代码、不 auto resolve、不 auto push、不 auto comment |
| `allowed-tools` | `Bash, Read, Glob, Grep` | `Bash, Read, Glob, Grep` |

证据：`skills/review-mr/SKILL.md`（frontmatter、Preconditions、数据获取、llmdoc 加载、6 维度、输出模板、约束段）；`skills/review-fixup/SKILL.md`（同前 + jq 检查 + 8 行 keyword 路由表 + 5 桶分类 + plan 模板）。

## 数据获取链

### review-mr（2 条命令，无 jq）

```bash
glab mr view <iid> --comments > /tmp/glab-mr-<iid>-view.txt
glab mr diff <iid>             > /tmp/glab-mr-<iid>-diff.patch
```

从 view 输出抽取：标题、描述、源/目标分支、现有评论、pipeline 状态。**不单独拉 commit**——commit 关键词从 view 已打印的内容里取，仅作为 topic 路由信号。

### review-fixup（3 阶段管线 + jq filter）

```bash
# 阶段 1：项目路径 URL 编码
PROJECT_PATH=$(glab repo view --output json | jq -r '.path_with_namespace')
PROJECT_PATH_ENCODED=$(echo -n "$PROJECT_PATH" | jq -sRr @uri)
# e.g. snow_workspace_group/sws-browser-extension → snow_workspace_group%2Fsws-browser-extension

# 阶段 2：discussions 全量（必须 --paginate）
glab api "projects/${PROJECT_PATH_ENCODED}/merge_requests/<MR>/discussions" \
  --paginate \
  > /tmp/glab-mr-<MR>-discussions.json

# 阶段 3：diff 上下文
glab mr diff <MR> > /tmp/glab-mr-<MR>-diff.patch
```

snow_dev_ai 过滤（`skills/review-fixup/SKILL.md` 内 jq 段，写到 `/tmp/glab-mr-<MR>-ai-findings.json`）：

```jq
[
  .[] | .notes[]
  | select(.author.username == "snow_dev_ai")
  | {
      discussion_id, note_id: .id, body,
      file: (.position.new_path // .position.old_path // null),
      new_line: .position.new_line,
      old_line: .position.old_line,
      resolved: (.resolvable and .resolved),
      created_at
    }
]
```

设计要点：
- 用 `discussions` 端点而非 `notes`——拿到的对象带 `position`，能映射 `file:line`。
- `--paginate` 是**约束级别要求**：所有 `glab api` 必须分页，否则漏数据。
- `resolved` 用 `(.resolvable and .resolved)`——非 resolvable 的系统 note 不会被算成 "已解决"。
- URL 编码必须用 `jq @uri`，**不允许** `sed` 手 hack。

边界处理：
- 数组空 → 输出"未发现 snow_dev_ai 记录，可能 AI 还没跑完"，结束。
- 全 resolved → 输出"所有 AI 评论已 resolved，无需处理"，除非用户明说仍要看。
- 部分 resolved → 默认只处理未解决，用户可 opt-in 看全部。

## llmdoc 加载策略

两个 skill 都先用 `[ -f llmdoc/index.md ] && echo "EXISTS" || echo "MISSING"` 探测被评审项目（cwd）下的 `llmdoc/`。

### review-mr：4 步策略

1. **Index** — `Read llmdoc/index.md`，按 Markdown 表 `| 文档 | 描述 |` 解析。
2. **Must** — `Glob llmdoc/must/*.md` → 全部 Read，作为硬约束。
3. **Topic-targeted** — 关键词信号来自三源：变更文件路径 / 变更符号名 / commit-message + MR description 关键词；与 index 描述匹配，加载最相关 3-5 篇。SKILL.md 给了 3 个示例：logger / 脱敏 / CI。
4. **Coding conventions** — 总是读 `llmdoc/reference/coding-conventions.md`（如存在），作为 readability 维度依据。

### review-fixup：4 步同 + 8 行明文路由表

关键词信号来自**所有 finding 涉及的文件路径 + 评论 body 关键词**。SKILL.md 内嵌了 8 个明文 keyword → doc 路由（`logger`、`脱敏/desensitization/mask`、`recording/录屏/rrweb`、`service worker/background/KeepServiceAlive`、`proxy/PAC`、`shared/KeepServiceAlive/indexedDB`、`CI/GitLab CI/飞书`、`DOM/性能/layout`）。`review-mr` 只有 3 个示例，`review-fixup` 有 8 行表格，**review-fixup 的 llmdoc 路由更结构化**。

### 缺失时降级

两个 skill 都 explicit fallback：跳过加载，**报告头必须写**"未加载项目专属规范（cwd 下未发现 `llmdoc/`）"。这是 `must/working-agreement.md` 列出的可见降级要求。

## 评审维度与桶分类

### review-mr 6 维度

| # | 维度 | 关键检查点 |
|---|---|---|
| 1 | Correctness | 是否实现 MR title/description；边界（空数组/null/undefined/并发）；错误处理；逻辑 bug / off-by-one |
| 2 | Security | OWASP Top 10；token/PII/password 泄漏到日志/error/URL；权限边界（content↔background messaging）；3rd-party CVE |
| 3 | Performance | O(N²)；不必要的 deep copy；DOM layout thrash；leak（unbound listener / 未清 timer / 大对象保留） |
| 4 | Readability | 命名清晰；one-function-one-thing；注释为何而非干啥；与 `llmdoc/reference/coding-conventions.md` 对齐 |
| 5 | Test coverage | unit/integration 是否补；non-happy-path 是否缺；既有测试是否同步更新 |
| 6 | Convention compliance | **仅当 llmdoc 存在时**；逐条 finding 引用 llmdoc 路径作为依据；llmdoc 缺失时该维度标 N/A |

每维度先写"无问题 / N 个问题"结论再列 finding。

### review-fixup 5 桶

| Tag | 含义 | Action |
|---|---|---|
| 🔴 valid bug | 真问题（逻辑错/安全漏洞/明确违规） | 必须修，给 diff |
| 🟡 valid improvement | 合理建议（命名/可读性/性能微优化） | 建议修，给 diff |
| ⚪ style preference | 风格分歧（分号/换行/命名风格） | 看团队约定，默认不修 |
| ⚪ false positive | AI 误报 | 不修，但要写明不修原因 |
| 🔵 uncertain | 缺信息无法判断 | 留给用户决策 |

per-finding 评估流程：
1. `Read` finding.file，聚焦 `new_line ± 20 行`；
2. 读相关 llmdoc 文档（如已加载）；
3. 判断 AI 提议在当前上下文是否合理；
4. 打 5 桶之一 tag；
5. 写 1-2 句 rationale。

判定标准：
- llmdoc `must/*.md` 违反 → 几乎必然 🔴。
- coding-conventions miss → 🟡（除非纯风格 → ⚪）。
- topic `architecture/` 设计原则冲突 → 🔴 或 🟡（按严重度）。
- llmdoc 缺失：理由仅基于代码上下文，报告头标注降级。

## 输出契约

### review-mr 评审报告

固定段落：
1. **Header**：标题、源→目标、review 时间戳、**"项目规范来源"行**（明确写 N 篇 llmdoc 已加载 / 未加载）。
2. **总体评价**：2-3 句。
3. **维度汇总表**：6 行，每行 `✓ 无问题` 或 `⚠ N 个问题`，convention 行可为 `N/A（无 llmdoc）`。
4. **Findings**：按 4 桶分组（🔴/🟡/🟢/📝），每条含 `file:line` + 标题 + 问题 + 依据（llmdoc 路径） + diff 建议。
5. **测试建议**：要补的测试场景列表。
6. **结论**：建议 ready / approve / 仍需修改。

### review-fixup 修复 plan

固定段落：
1. **Header**：`# AI Review Fix Plan: MR !<iid>`、项目路径、评估时间戳、llmdoc 加载条数。
2. **评估摘要**：4 计数器（N total / 🔴=A / 🟡=B / ⚪=C / 🔵=D）。
3. **🔴 必须修复**：per-item `file:line — 标题`、AI 评论原文 / 摘要、评估（同意 + 理由）、依据（llmdoc 路径或通用最佳实践）、修复（fenced **diff 块**）。
4. **🟡 建议改进**：同 per-item 形态。
5. **⚪ 可忽略项**：折叠成表 `| 文件:行 | AI 提议 | 不修原因 |`。
6. **🔵 待决策**：per-item 含 AI 评论、为什么不能立刻判断、需要你确认的问题。
7. **推荐执行顺序**：5 步（修 🔴 → 修 🟡 → 推荐 commit message `fix(review): apply AI review feedback for MR !<iid>` → 跑 `pnpm typecheck && pnpm test`（或仓库实际命令） → push 触发 GitLab AI 重评 → 必要时复跑 `/glab:review-fixup <iid>`）。
8. **Optional GitLab follow-ups**：可手动 mark resolved，但 skill **不会** 调 `glab discussion resolve`。

## 安全边界

两个 skill 都是 plan-only / advisory：

- ❌ 不修改代码（`allowed-tools` 不含 `Edit`/`Write`）。
- ❌ 不 auto resolve discussion。
- ❌ 不 auto push。
- ❌ 不 auto comment 到 GitLab。
- ✅ 只产出 Markdown 报告/plan，用户自行决定是否应用。

## 被评审项目的 llmdoc 期望结构（消费契约）

review skill 假定被评审项目 cwd 下的 `llmdoc/` 至少包含：

| 路径 | 用途 | 缺失影响 |
|---|---|---|
| `llmdoc/index.md` | 探测点 + 路由索引（Markdown 表 `\| 文档 \| 描述 \|`） | 不存在 → 整个 llmdoc 加载链跳过，报告头降级标注。 |
| `llmdoc/must/*.md` | 硬约束 | 缺失则 review-mr 第 6 维度（convention）退化为 N/A；review-fixup 5 桶失去最强 🔴 判定标准。 |
| `llmdoc/reference/coding-conventions.md` | readability 维度依据 | 不存在则 review-mr readability 检查只能基于通用知识。 |
| `llmdoc/architecture/*.md`、`llmdoc/guides/*.md`、`llmdoc/reference/*.md` | topic 路由目标 | 影响 review 的"项目专属性"——index 里有路由词但目标不存在时，路由失效但不影响主流程。 |

> 提示：**本仓库自身的 `llmdoc/` 目前没有 `reference/coding-conventions.md`**，意味着如果用 `review-mr` 评审本仓库自己的代码，readability 维度会降级。详见 `memory/doc-gaps.md`。

## 已知 gap

- `snow_dev_ai` username 硬编码——bot 改名时 jq filter 与 README 都要同步改。
- `review-mr` 没有给 `glab mr view --comments` 加 `--paginate`，长评论列表可能截断。
- `review-fixup` 推荐 `pnpm typecheck && pnpm test`，但无 per-project 检测逻辑。
- review skill 工作于 whole-MR 范围，未提供按文件/按 hunk 缩窄的入参。
- llmdoc 加载逻辑在两个 SKILL.md 内近乎复制粘贴——索引格式变更需同步。

## Related Docs

- `architecture/skill-system.md` — 共享约定背景。
- `must/working-agreement.md` — plan-only / 降级可见 等强约束。
- `memory/doc-gaps.md` — 已知文档/实现 gap。
- `skills/review-mr/SKILL.md` / `skills/review-fixup/SKILL.md` — 一手定义。
