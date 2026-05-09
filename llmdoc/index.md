# llmdoc 全局导航

`glab` 在 Claude Code / Codex 双生态下的稳定文档地图。每条任务先来这里查"该读哪份"，再深入文档本体。

## 类别用途

| 目录 | 用途 |
|---|---|
| `must/` | 每次启动必读的最小知识。项目身份、跨 skill 强约束、文档路由。 |
| `overview/` | 项目身份与边界（做什么 / 不做什么）。 |
| `architecture/` | skill 互联图、共享约定、不变量、review pipeline 双路。 |
| `reference/` | manifest 字段语义、命名映射规则等稳定查表型事实。 |
| `guides/` | 单一工作流操作手册（一文档一流程）。 |
| `memory/` | 历史过程记忆。`reflections/` 由 reflector 拥有；`decisions/`、`doc-gaps.md` 由 recorder 拥有。 |

## 文档清单

### must/
- `must/project-basics.md` — Claude/Codex 双入口身份、8 个 skill 一句话索引、外部依赖、仓库当前状态。
- `must/working-agreement.md` — 跨 skill 共享的强约束（不动用户环境、不写 git、不 auto-resolve 等）与降级策略。
- `must/doc-routing.md` — 任务到文档的快速路由表。

### overview/
- `overview/project-overview.md` — Identity / Boundaries / Major Areas，强调"封装 + AI review 二次评估"的产品立场。

### architecture/
- `architecture/skill-system.md` — 8 个 skill 的角色分组、cross-skill 推荐图、frontmatter 三件套与 Preconditions 模板、临时文件命名、严重度词汇、不变量与失败模式。
- `architecture/review-pipeline.md` — `review-mr` 与 `review-fixup` 双路评审管线：触发 / 数据获取 / llmdoc 加载策略对比 / 评审维度与桶分类 / 输出契约 / 安全边界 / 被评审项目的 llmdoc 期望结构。

### reference/
- `reference/plugin-manifest.md` — `.claude-plugin/` / `.codex-plugin/` / `.agents/plugins/marketplace.json` / SKILL.md frontmatter 字段语义；命令前缀映射规则；分发目录区分；触发短语机制。
- `reference/coding-conventions.md` — SKILL.md 结构、命名规范、中文 Markdown 输出风格、emoji 词汇、Bash/glab/jq one-liner 规范、反模式列表（`review-mr` readability 维度依据）。

### guides/
- `guides/add-new-skill.md` — 端到端流程：选名 → 三件套 frontmatter → Preconditions 模板 → /tmp 决策 → 接入推荐图 → 验证。
- `guides/release-and-distribute.md` — Claude Code 与 Codex 的本地迭代、首次发布、团队升级路径；含 `marketplace update`、plugin.json 版本号与 Codex 接入提示。

### memory/
- `memory/doc-gaps.md` — 已知文档/实现 gap 与建议补救（跨工具兼容声明矛盾、`/tmp` 跨平台、`auth status --hostname` 不一致、Codex marketplace 路径对齐等）。
- `memory/decisions/` — 重要决策记录（暂空）。
- `memory/reflections/` — reflector 输出（含 Codex plugin 配置落库后的 README / llmdoc 对齐反思）。

## 路由提示

- 改某个 skill → 先读 `architecture/skill-system.md` 的"共享约定"段。
- 改 review-mr / review-fixup → 先读 `architecture/review-pipeline.md`。
- 改 `.claude-plugin/plugin.json` / `.claude-plugin/marketplace.json` / `.codex-plugin/plugin.json` / `.agents/plugins/marketplace.json` / SKILL.md frontmatter → 先读 `reference/plugin-manifest.md`。
- 加新 skill → 先读 `guides/add-new-skill.md`。
- 安装 / 分发 / 发版 → 先读 `guides/release-and-distribute.md`。
- 失败/混乱 → 翻 `memory/reflections/` 与 `memory/doc-gaps.md`。

> 启动顺序见 `startup.md`，本文不重复。
