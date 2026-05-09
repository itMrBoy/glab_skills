# Project Overview — `glab` plugin

## Identity

一个以 Claude Code 为主、同时提供 Codex plugin 元数据的 skill 仓库，把 snow 团队最常用的 GitLab 操作（看 MR / 创 MR / 看 CI / 看 diff / 看领先 commit）和 AI review 工作流封装成 8 个自然语言可触发的 skill。Claude 侧通过 `/glab:<name>` 调用；Codex 侧复用同一套 `skills/` 目录与 plugin manifest。

- Plugin name：`glab`
- Marketplace name：`snow-glab-marketplace`
- 默认 GitLab 实例：`git.snowsse.cn`
- 目标平台：Windows + PowerShell（教程层），但执行层依赖 bash shell（`/tmp/...` 路径）。

## Boundaries

### 做什么

- **封装 glab 高频操作**：mr / mr-create / mr-diff / pipeline / commits-ahead / setup —— 把"先 `glab mr list --source-branch ...`，再 `glab mr view <iid>`，再总结"这种几步流程压成一句自然语言。
- **AI review 二次评估**：
  - `review-mr` 主动评审任意 MR，结合被评审项目 cwd 下的 `llmdoc/` 做 6 维度判断。
  - `review-fixup` 解析 GitLab `snow_dev_ai` 机器人留下的 review 评论，逐条评估合理性后产出修复 plan（不改代码）。
- **诊断 + 教程化引导**：`setup` 用只读检测判定四态（未装 / 已装未认证 / token 失效 / 已就绪），按态输出整段中文 PowerShell 教程让用户手动执行。

### 不做什么

- ❌ 不修改用户 shell / 环境变量 / token / GitLab 配置文件。
- ❌ 不替用户跑 `winget` / `scoop` / `choco` / `glab auth login`。
- ❌ 不 `git push` / `git commit` / `git rebase` / 不 force push。
- ❌ 不 auto resolve / auto comment / auto retry / auto merge。
- ❌ 不替代人类决策：`mr-create` 强制人工确认标题描述；`review-fixup` 只产 plan。
- ❌ 不替团队成员维护其它项目的 `llmdoc/`——review skill 是消费者，不是写入者。

## Major Areas

### A. 基础查询与创建（5 个 skill）

`mr` / `mr-create` / `mr-diff` / `pipeline` / `commits-ahead`。共享 git 当前分支推断模式 + glab auth 防御 + 中文 Markdown 输出。`mr` 是中枢推荐器，会按 MR 状态推荐 mr-diff / pipeline / review-fixup。

### B. 环境引导（1 个 skill）

`setup`。是其它 4 个 glab 依赖 skill 的统一 fallback——auth 失败 → 调 setup → 终止本流程。

### C. AI review 双路（2 个 skill）

`review-mr` 与 `review-fixup`。这是该插件相对原生 glab 的差异化价值：把 GitLab AI 机器人的产出和项目专属 `llmdoc/` 知识结合，给出"哪些必修、哪些可忽略、哪些待决策"的结构化输出。详见 `architecture/review-pipeline.md`。

### D. Plugin 骨架与分发

`.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` + `.codex-plugin/plugin.json` + `.agents/plugins/marketplace.json` + `README.md`。当前仓库在同一套 `skills/` 上同时维护 Claude 与 Codex 的接入元数据；其中 Codex marketplace 路径仍需与最终 plugin 根目录保持一致。详见 `reference/plugin-manifest.md`。

## 关键设计立场

1. **教程化而非自动化**——任何会改用户机器/账号的操作都返工给用户（教程文本），保留人类决策权。
2. **plan-only 评审**——review skill 只读、不动代码、不动 GitLab 状态。
3. **降级可见**——llmdoc 缺失时 review skill 不静默降级，必须在报告头标注"未加载项目专属规范"。
4. **约定优于配置**——SKILL.md frontmatter 仅 3 字段（`name` / `description` / `allowed-tools`），命令名靠目录名 + plugin name 自动拼接。
5. **同一 skill 源，多入口接入**——Claude Code 走 plugin/marketplace，Codex 走 `.codex-plugin` 或 skill path；兼容性按语义保证，不承诺工具 ID 完全一致。
