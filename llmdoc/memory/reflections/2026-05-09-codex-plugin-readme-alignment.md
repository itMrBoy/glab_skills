# Reflection — Codex plugin 配置落库后的 README / llmdoc 对齐

## 背景

本次任务为仓库补充了 Codex 侧的 plugin 元数据，并据此更新了 `README.md`：

- 新增 `.codex-plugin/plugin.json`
- 新增 `.agents/plugins/marketplace.json`
- 将 README 中的 Codex 说明从“未来适配 / 完全跨工具兼容”改成“已提供配置 / 语义可移植”

## 这次暴露出的文档问题

1. **README 对 Codex 状态描述滞后**
   - 之前 README 仍写“Codex CLI（未来适配）”，但仓库已经有可被 Codex 消费的 manifest。
   - 这类“实现已变，README 还停留在旧口径”的问题会误导第一次安装的使用者。

2. **llmdoc 的项目状态快照过时**
   - `must/project-basics.md`、`guides/release-and-distribute.md`、`memory/doc-gaps.md` 里仍保留“main 无 commit”“GitHub 路径不可用”等初始化时期结论。
   - 这类快照型文档如果不随里程碑（首次 commit / 首次 push / 新生态接入）回写，会把一次性的历史事实误写成长期事实。

3. **“跨工具兼容”要区分语义层与协议层**
   - `SKILL.md` 里的 `Read` / `Glob` 等工具名仍带 Claude Code 风格。
   - 更准确的说法应是：**核心工作流可复用，具体工具能力由宿主 agent 映射**。

## 这次应该被提升为稳定知识的内容

- 仓库已同时维护 Claude Code 与 Codex 两套 plugin 入口：
  - Claude：`.claude-plugin/plugin.json`、`.claude-plugin/marketplace.json`
  - Codex：`.codex-plugin/plugin.json`、`.agents/plugins/marketplace.json`
- Codex 场景不应再只写“把 `skills/` 加到搜索路径”，而应优先说明已有 plugin / marketplace 入口，再把 skill-path 方案当兜底。
- 涉及“仓库当前状态”的文档必须在首次 commit / push 后立即回写，避免历史快照长期滞留。

## 下次遇到类似任务的操作提醒

- 只要新增了 manifest、marketplace、插件入口或分发方式，除了 README，还应同步检查：
  - `must/project-basics.md`
  - `reference/plugin-manifest.md`
  - `guides/release-and-distribute.md`
  - `memory/doc-gaps.md`
- 若 README 中出现“完全兼容”“未来适配”这类强表述，改实现时要优先验证这些措辞是否还成立。
