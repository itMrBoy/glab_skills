# 文档路由

任务 → 文档的最快查表。先 routing，再深入。

## 改 Skill

| 改动对象 | 先读 |
|---|---|
| 任意 SKILL.md frontmatter / Preconditions / 输出格式 | `architecture/skill-system.md`（共享约定段） |
| `review-mr` / `review-fixup` 内的评审逻辑或 llmdoc 加载策略 | `architecture/review-pipeline.md` |
| `setup` 的诊断分支 / 教程文案 | `architecture/skill-system.md`（不变量与失败模式段） |
| `mr-create` 的 default target / draft 行为 | `must/working-agreement.md` + `architecture/skill-system.md` |

## 改 Manifest / 安装链路

| 改动对象 | 先读 |
|---|---|
| `.claude-plugin/plugin.json` 任一字段 | `reference/plugin-manifest.md` |
| `.claude-plugin/marketplace.json` 任一字段 | `reference/plugin-manifest.md` |
| README 安装/升级片段 | `reference/plugin-manifest.md` + `memory/doc-gaps.md`（确认是否触发已知 gap） |

## 增加 / 重构

| 任务 | 先读 |
|---|---|
| 加新 skill | `guides/add-new-skill.md` |
| 升级 / 分发 plugin | `guides/release-and-distribute.md` |
| 引入新 manifest 字段（如 `argument-hint`、`hooks`） | `reference/plugin-manifest.md` 然后扩展 |

## 出错 / 混乱

| 现象 | 先读 |
|---|---|
| 用户报告"装不上 / 拉不到" | `memory/doc-gaps.md`（README 拉取路径目前不可用） |
| 跨平台路径报错（cmd 下 `/tmp` 解析失败等） | `memory/doc-gaps.md`（`/tmp` 跨平台 gap） |
| `glab auth status` 通过但 MR 命令失败 | `memory/doc-gaps.md`（hostname 不一致 gap） |
| 不清楚为什么这么设计 | `memory/reflections/`（暂空） |
| 重大架构选择背后的 why | `memory/decisions/`（暂空） |
