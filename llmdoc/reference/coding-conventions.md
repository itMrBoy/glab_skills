# Coding Conventions

本仓库内**所有 SKILL.md 与 llmdoc 文档**遵守的最小命名 / 结构 / 格式约定。`review-mr` 在 readability 维度会加载本文件作为依据；改 SKILL.md 时按本表自检。

## SKILL.md 结构

按下列顺序组织（参考现有 8 个 SKILL.md）：

1. **frontmatter** — `name` / `description` / `allowed-tools` 三件套，无其它字段。
2. **`# /glab:<name> — <一句话用途>`** — 一级标题。
3. **`## Preconditions`** — 仅当依赖外部 CLI（glab、jq）时写；按 `must/working-agreement.md` "失败时降级"模板。
4. **`## 主流程`** — 分 step 写，每 step 一段 bash 代码块 + 解析说明。
5. **`## 输出`** 或 **`## 中文总结输出`** — 固定中文 Markdown 模板。
6. **`## 失败模式`** — 可预见的边界与处理（detached HEAD、空数组、auth 失败等）。
7. **`## 与其它 skill 的关系`** — cross-skill 推荐说明。

允许省略 4-7 中的小节，但**不允许**重排顺序（让读者一眼定位）。

## 命名规范

| 对象 | 规则 | 例子 |
|---|---|---|
| Skill 目录名 | 短横分隔小写 | `mr-create`、`commits-ahead`、`review-fixup` |
| Skill frontmatter `name` | **必须等于**父目录名 | `name: mr-create` |
| 临时文件 | `/tmp/glab-mr-<iid>-<role>.<ext>` POSIX 路径 | `/tmp/glab-mr-42-diff.patch` |
| Bash 变量 | UPPER_SNAKE_CASE | `PROJECT_PATH_ENCODED` |
| 中文标题 | 用空格分隔英文与中文（视觉舒适） | `# /glab:mr — 查看当前分支的 MR` |

## 中文 Markdown 输出风格

适用于 skill 的最终输出（不是 SKILL.md 自身）：

- **一律中文**。技术术语保留原形（`pipeline`、`MR`、`draft`、`merge_request`）。
- **段落标题层级**：报告主标题 `##`，子段 `###`，再深用列表。
- **状态用 emoji** 而非纯文字：状态、严重度、判定结果都用统一 emoji（见下）。
- **链接**：直接给 `<web_url>` 行，不用 `[文本](url)` 包装（用户会复制粘贴到浏览器）。
- **不超 N 字总结要遵守**：如 `mr-diff` 总结 ≤500 字，超出时合并文件类别。

## 严重度 emoji 词汇（统一）

| Emoji | 含义 | 使用位置 |
|---|---|---|
| 🔴 | 必须修复 / valid bug / 流水线失败 | review-mr / review-fixup / pipeline |
| 🟡 | 建议改进 / valid improvement / 警告 | review-mr / review-fixup |
| 🟢 | 加分项 / 通过 / Nice to Have | review-mr / pipeline |
| ⚪ | 风格分歧 / false positive | review-fixup |
| 🔵 | 待决策 / uncertain（缺信息无法判断） | review-fixup |
| 📝 | 风格分歧汇总表标题 | review-mr |

不要引入 🟣 / 🟠 等新 emoji 而不更新本表。

## Bash one-liner 风格

- **JSON 输出统一加 `--output json`**：`glab mr list --source-branch <b> --output json`。
- **API 调用必须 `--paginate`**：`glab api ... --paginate`。漏 `--paginate` 是已知 gap（详见 `memory/doc-gaps.md`）。
- **URL 编码用 `jq -sRr @uri`**，**禁止** `sed` 手 hack。
- **临时文件输出到 `/tmp/glab-mr-<iid>-*`**：通过 `>` 重定向，再用 `Read` 工具加载，避免 stdout 灌满上下文。
- **不在 SKILL.md 里 `cd <project-root>`**：Claude Code session cwd 已在项目根；多余的 `cd` 触发权限提示。
- **检查命令存在**：`glab --version`、`jq --version`，不要用 `which`（跨平台不可靠）。
- **避免 `grep` / `cat` / `find`**：模型应改用 Grep / Read / Glob 工具；SKILL.md 例子里写 bash 是给 Bash 工具执行的真实命令，**不是**让模型抄。

## glab 子命令风格

- **新版优先**：`glab ci ...`、`glab mr view --comments`、`glab api ...`。
- **旧版回退仅作 fallback**：`pipeline` 在新命令失败时尝试 `glab pipeline ...`，但**不要**默认走旧版。
- **hostname 显式声明**：长期目标是所有 `glab auth status` 都带 `--hostname git.snowsse.cn`。当前 setup 已显式，其它 skill 未带是已知 gap（`memory/doc-gaps.md` G4）。

## jq 风格

- 多行 jq filter 写在 fenced \`\`\`jq 代码块里，不要塞进 `-r '...'` 单行字符串。
- 复合判断用 `(.resolvable and .resolved)` 这种括号显式分组，避免读者解析歧义。
- 投影对象时**字段名按数据流层级**排（外层 ID → 嵌套字段）：

  ```jq
  {
    discussion_id, note_id: .id, body,
    file: (.position.new_path // .position.old_path // null),
    new_line: .position.new_line,
    old_line: .position.old_line,
    resolved: (.resolvable and .resolved),
    created_at
  }
  ```

## 错误信息风格

- **面向用户**：用第二人称中文（"当前不在分支上，无法定位 MR"），不是英文 stack trace。
- **给出下一步**：错误后必须告诉用户做什么（"请运行 /glab:setup"、"请指定 MR iid"）。
- **不暴露 token / 路径敏感信息**：错误展示前剥掉敏感字段。

## 文档引用格式

跨 llmdoc / 跨仓库引用按 `must/working-agreement.md` 与 `architecture/skill-system.md` 的现行写法：

- **同 llmdoc 内引用**：`架构详见 architecture/skill-system.md`（不带 `llmdoc/` 前缀，不带反引号也行）。
- **代码或路径**：用反引号 `skills/mr/SKILL.md`。
- **代码 + 符号 / 章节**：`skills/mr/SKILL.md` (`Preconditions` 段)，**默认不写行号**——行号只在证明非显然行为时才加。

## 反模式（不要这样写）

| ❌ 反模式 | ✅ 正确做法 |
|---|---|
| 在 SKILL.md frontmatter 写 `tools:` | 用 `allowed-tools:` |
| 用 `git push --force` 修复历史 | 不 force push；改用新 commit |
| `glab discussion resolve` 自动 resolve | 只输出修复 plan，不动 GitLab 状态 |
| 在 review-fixup 里调 Edit / Write 改代码 | plan-only，让用户自己 apply |
| `glab api ...` 不加 `--paginate` | 永远 `--paginate` |
| `cd C:\...项目根` 后再跑 git | 直接跑 git，session cwd 已对 |
| 输出英文报告或大段 stack trace | 一律中文 Markdown |

## Related Docs

- `architecture/skill-system.md` — 共享约定。
- `must/working-agreement.md` — 强约束底线。
- `architecture/review-pipeline.md` — review 输出契约的具体结构。
- `guides/add-new-skill.md` — 新 skill 适用本文件。
