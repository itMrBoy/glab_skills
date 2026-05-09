# 启动阅读顺序

每次进入新会话先按本顺序读完 must 三件套，再按需升级。

## 必读（按序）

1. `llmdoc/must/project-basics.md` — 知道这是什么 plugin、有哪些 skill、目标平台与仓库当前状态。
2. `llmdoc/must/working-agreement.md` — 跨 skill 强约束与失败时降级规则。
3. `llmdoc/must/doc-routing.md` — 任务到深文档的快速路由。

## 升级阅读触发

| 当你要做 | 立即升级阅读 |
|---|---|
| 改任意 SKILL.md | `architecture/skill-system.md` |
| 改 `review-mr` / `review-fixup` 或讨论评审策略 | `architecture/review-pipeline.md` |
| 改 `.claude-plugin/plugin.json` / `marketplace.json` 或 SKILL.md frontmatter | `reference/plugin-manifest.md` |
| 解释项目身份、边界、与外部用户对话 | `overview/project-overview.md` |
| 评估某个改动是否值得做 / 撞到已知问题 | `memory/doc-gaps.md` + `memory/reflections/` |

> 全局类别地图见 `index.md`，本文不重复。
