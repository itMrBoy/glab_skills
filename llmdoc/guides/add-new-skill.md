# How to Add a New Skill

新增 skill 的端到端流程。本插件 8 个 skill 全部遵守同一组约定，新 skill 必须接入这套约定才能与 `mr` 中枢推荐器、`setup` fallback、`/tmp` 临时文件协议无缝协作。

## 1. 准备

- 已经按 `llmdoc/startup.md` 读完 must 三件套，对 `architecture/skill-system.md` 的共享约定有清晰认识。
- 想清楚 skill 的**单一职责**：一个 skill 只做一件事。多职责时拆成多个 skill，用 cross-skill 推荐串起来（参考 `mr` 推 `mr-diff` / `pipeline` / `review-fixup` 的写法）。
- 想清楚是否真的需要新 skill：能用现有 skill 组合解决就不开新坑。

## 2. 选 skill 名

- 短横分隔（`mr-create`、`commits-ahead`），不要驼峰，不要下划线。
- 名字会同时出现在三处：`skills/<name>/`、`SKILL.md` frontmatter `name`、用户输入的 `/glab:<name>`。
- 避免与现有 8 个 skill 重名（`setup` / `mr` / `mr-create` / `mr-diff` / `pipeline` / `commits-ahead` / `review-mr` / `review-fixup`）。

## 3. 创建目录与 SKILL.md

```bash
mkdir -p skills/<name>
touch skills/<name>/SKILL.md
```

frontmatter 三件套必须齐全（详见 `reference/plugin-manifest.md`）：

```yaml
---
name: <name>                 # 必须 = 父目录名
description: <一句话用途>. Use when user says <中英自然语言触发短语>.
allowed-tools: Bash[, Read[, Glob[, Grep]]]
---
```

`description` 末尾的 "Use when user says ..." 是模型识别 slash 之外自然语言入口的依据，不要省。

## 4. 决定 allowed-tools

按下面表格选最小集（参考 `architecture/skill-system.md` 的 allowed-tools 分布）：

| 你要做的事 | 加什么 |
|---|---|
| 跑 `glab` / `git` / `jq` 等 CLI | `Bash` |
| 读 `/tmp` 临时文件或 cwd 下其它已知路径 | `Read` |
| 遍历 `llmdoc/` 或被评审项目目录 | `Glob` |
| 在被评审代码里搜符号 | `Grep` |

**不要写** `Edit` / `Write` / `NotebookEdit`——本插件至今没有任何 skill 修改用户文件，加这些工具会破坏 `must/working-agreement.md` 的"plan-only"边界。

## 5. 决定是否要 glab 前置检查

如果会调 `glab` 任意子命令，**必须**写 Preconditions 段，按 4 个基础 skill 共用的模板：

```markdown
## Preconditions

依赖 `glab` CLI 已安装并完成 git.snowsse.cn 认证。

执行前先跑：

\`\`\`bash
glab auth status
\`\`\`

如果失败（未安装 / 未认证 / token 过期 / command not found）：

1. 调用 `glab:setup` skill 拿到诊断结果与教程
2. 把教程**完整原样**输出给用户
3. 在末尾追加："完成上述步骤后请重新运行 /glab:<name>"
4. **终止本 skill 执行**（不要继续）
```

例外情形：
- 纯本地 git 操作（参考 `commits-ahead`）→ 不需要 glab 检查，但要在 SKILL.md 显式写"本 skill 不依赖 glab"，避免读者误以为漏写。
- 需要 `jq` → 在 Preconditions 段**额外**加 `jq --version` 检查，失败提示 `winget install -e --id jqlang.jq` 后终止（参考 `review-fixup`）。

## 6. 决定是否要 /tmp 临时文件

如果输出量大（patch / discussion JSON），不要直接 stdout 灌进上下文。落到 `/tmp/glab-mr-<iid>-<role>.<ext>`，再用 `Read` 工具按需加载。命名规范：

| 内容 | 后缀 |
|---|---|
| MR diff/patch | `-diff.patch` |
| MR view 文本输出 | `-view.txt` |
| GitLab API discussions | `-discussions.json` |
| jq 处理后的中间结果 | `-<purpose>.json` |

注意：`/tmp` 是 POSIX 路径，依赖 git-bash / WSL 提供映射（详见 `memory/doc-gaps.md` G3）。Windows 原生 cmd / PowerShell 没有 `/tmp`，但 Claude Code 默认 shell 是 bash 所以本地能跑。**当前不要改这个约定**，新 skill 沿用同一前缀。

## 7. 写主流程

按下面顺序组织 SKILL.md 正文（参考 `mr/SKILL.md` 与 `review-fixup/SKILL.md`）：

1. 标题：`# /glab:<name> — <一句话用途>`
2. `## Preconditions`（如适用）
3. `## 主流程`：分 step 写，每 step 给一段 Bash 代码块 + 解析说明
4. `## 输出` 或 `## 中文总结输出`：固定的中文 Markdown 模板
5. `## 失败模式`（可选）：列举可预见的边界与处理
6. `## 与其它 skill 的关系`：什么条件下推用户去哪个 skill

输出要求：
- 一律中文 Markdown。
- 复用统一的严重度 emoji（🔴/🟡/🟢/⚪/🔵/📝），含义见 `must/working-agreement.md`。
- 别在 SKILL.md 内写 `/glab:<name>` 命令名作为 frontmatter 字段——命令名是 plugin 自动拼的（详见 `reference/plugin-manifest.md`）。

## 8. 接入 cross-skill 推荐图

新 skill 至少要回答：
- 在它内部，发现什么状态时应当推用户去其它 skill？
- 在哪些**已有** skill 的"操作建议"段，应当增加对它的推荐？

如果新 skill 是用户首次接触某状态的入口（例如又一个查询类），考虑改 `mr/SKILL.md` 的"给操作建议"段加一行回指。改完同步更新 `architecture/skill-system.md` 的 ASCII 推荐图。

## 9. 验证

```powershell
# 让 marketplace 重新加载
claude plugin marketplace update snow-glab-marketplace
# 重启 Claude Code
# 在新会话里测试
/glab:<name>
```

也用自然语言触发 description 末尾声明的短语，确认模型能不靠 slash 命令识别意图。

## 10. 更新文档

- 在 `must/project-basics.md` 的 "8 个 Skill 索引" 表加一行（**注意：表标题的"8 个"也要改**）。
- 在 `architecture/skill-system.md` 的角色分组、推荐图、命令速查表三处补一行。
- 如果 skill 引入了新外部依赖（除 glab/git/jq 外），加到 `must/project-basics.md` 的"关键外部依赖"表。
- 本 guide 末尾的 "已知失败点" 段如果你新踩到坑，反向补进来。

## 已知失败点

- **目录名 ≠ frontmatter `name`**：slash 命令将无法被发现。三处必须同步。
- **frontmatter 里写 `tools:` 而非 `allowed-tools:`**：Anthropic skill 标准字段名是 `allowed-tools`，写错会让权限声明失效。
- **忘记 Preconditions**：用户没装 glab 会直接看到 raw error，违反 working-agreement 的"失败时降级"。
- **静默 fallback**：例如 llmdoc 缺失时不在报告头标注——违反 working-agreement 第 36 行。降级必须**用户可见**。
- **Bash 代码块用 `cd <project-root>`**：Claude Code session cwd 已在项目根，再 `cd` 触发权限提示；用绝对路径或省略 `cd`。

## Related Docs

- `architecture/skill-system.md` — 8 个 skill 全景与推荐图。
- `reference/plugin-manifest.md` — frontmatter 字段语义、命令命名映射。
- `must/working-agreement.md` — 失败降级与输出统一约定。
- `reference/coding-conventions.md` — SKILL.md / Markdown / Bash 代码风格。
- `guides/release-and-distribute.md` — 改完后怎么让团队用上。
