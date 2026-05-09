# 已知文档 / 实现 Gap

发现于 llmdoc init 调查（2026-04-30）。每条标注证据来源 + 建议补救。优先级仅作参考。

## G1. README "从 GitHub 拉取安装" 路径目前不可用 — 高 — **已解决 (2026-05-09)**

- **证据**：`git log --oneline --decorate -1` 显示 `HEAD -> main, origin/main, origin/HEAD`，说明远端默认分支已不再是空仓库。
- **影响**：初始化阶段这个问题会阻断团队成员从 GitHub 安装；当前已不再是主问题。
- **解决**：README 顶部已改为条件化提醒（“若当前改动尚未 push，则先用本地路径安装”），不再把仓库描述成“尚未发布”。

## G2. README "完全跨工具兼容" 与 SKILL.md 内 Claude Code 专属工具表述冲突 — 中 — **部分缓解 (2026-05-09)**

- **证据**：`skills/mr-diff/SKILL.md`、`skills/review-mr/SKILL.md`、`skills/review-fixup/SKILL.md` 正文多处直接以中文要求模型"用 Read 工具读取"、`Read llmdoc/index.md`、`Glob llmdoc/must/*.md`——这些是 Claude Code 内置 PascalCase 工具名，Codex CLI 工具空间不同。
- **影响**：Codex CLI 下严格意义上不能直接复用，得靠 LLM 把"用 Read 工具"翻译成等价文件读取（鲁棒性兜底而非协议兼容）。
- **当前状态**：README 已把“完全跨工具兼容”改为“语义可移植，工具能力需宿主 agent 映射”。
- **剩余补救**：
  - 在 SKILL.md 里继续把工具表述改成更中性的“读取 `<path>`”。
  - 若未来要做协议层兼容，再为 Codex 单独维护等价工具名或适配层。

## G3. `/tmp` POSIX 路径与 Windows 安装说明的隐式依赖 — 中

- **证据**：`mr-diff` / `review-mr` / `review-fixup` 都把临时文件写到 `/tmp/glab-mr-<iid>-*`（POSIX）；`setup` 教程通篇假定 Windows + PowerShell。Windows 原生 cmd / PowerShell 没有 `/tmp` 目录，依赖 git-bash / WSL / Cygwin 提供 `/tmp` 映射。
- **影响**：用户若在 Windows 用 cmd / 原生 PowerShell 启动 Claude Code，临时文件路径将解析失败。Claude Code 默认 shell 通常是 bash 所以暂未爆，但**未在任何 SKILL.md 或 README 显式声明**。
- **建议补救**：
  - README 加"**前置要求：Windows 用户需安装 Git for Windows，让 Claude Code 拥有 bash shell**"。
  - 或在 `setup` 教程的"环境检查"段加 shell 探测。
  - 长期：把 `/tmp` 替换为 `$TMPDIR`/`%TEMP%` 兼容方案，但工程量大。

## G4. `glab auth status --hostname` 在 setup 与其它 skill 不一致 — 中

- **证据**：`skills/setup/SKILL.md` 用 `glab auth status --hostname git.snowsse.cn`；`skills/mr` / `mr-create` / `mr-diff` / `pipeline` / `review-mr` / `review-fixup` 的 Preconditions 都只写 `glab auth status`（不带 hostname）。
- **影响**：用户配置了多 host（如同时认证了 gitlab.com 和 git.snowsse.cn）时，`glab auth status` 检查可能因任意一个 host 通过而放行，但实际 MR 命令仍会因目标 host 未认证失败。setup 与运行期的认证语义不一致。
- **建议补救**：
  - 把所有 skill 的前置检查统一为 `glab auth status --hostname git.snowsse.cn`。
  - 或把 hostname 抽到一个共享常量文档（如 `must/project-basics.md`），每个 skill 引用。

## G5. Codex marketplace `source.path` 与当前仓库布局不一致 — 中

- **证据**：仓库存在 `.codex-plugin/plugin.json` 于 repo 根，但 `.agents/plugins/marketplace.json#plugins[0].source.path = "./plugins/glab"`；工作树中不存在 `plugins/glab/` 目录。
- **影响**：README 若直接声称“可直接复用本地 Codex marketplace”，会误导读者；Codex 通过 marketplace 接入时可能找不到 plugin 根。
- **建议补救**：
  - 若决定“repo 根即 plugin 根”，把 `source.path` 改成与仓库实际布局一致的路径。
  - 若决定遵循 `plugins/<name>/` 结构，则把 `.codex-plugin/plugin.json` 移到对应目录并同步更新 README / llmdoc。

## G6. 缺 `guides/add-new-skill.md` — 中 — **已解决 (2026-04-30)**

- **证据**：`must/doc-routing.md` 路由表里引用了它，但当前 `llmdoc/guides/` 为空。
- **解决**：`llmdoc/guides/add-new-skill.md` 已写入；`doc-routing.md` "(待写)" 标记已移除。

## G7. 缺 `guides/release-and-distribute.md` — 中 — **已解决 (2026-04-30)**

- **证据**：`must/doc-routing.md` 路由表引用，但 `llmdoc/guides/` 为空。
- **解决**：`llmdoc/guides/release-and-distribute.md` 已写入，覆盖本地迭代 / 首次发布 / 团队升级 / Codex CLI 四种场景；`doc-routing.md` "(待写)" 标记已移除。

## G8. 缺 `reference/coding-conventions.md`（自食狗粮 gap）— 中 — **已解决 (2026-04-30)**

- **证据**：`skills/review-mr/SKILL.md` 把 `llmdoc/reference/coding-conventions.md` 列为 readability 维度的固定加载文档；本仓库自己的 `llmdoc/reference/` 当时只有 `plugin-manifest.md`。
- **解决**：`llmdoc/reference/coding-conventions.md` 已写入，覆盖 SKILL.md 结构 / 命名规范 / 中文 Markdown 输出 / emoji 词汇 / Bash 与 glab 与 jq 风格 / 反模式表。`review-mr` 用本仓库自身做评审时 readability 维度不再降级。

## 次要观察（暂不列为 gap，记录备查）

- `mr-create` 未防御 source branch 未 push 的情况——`glab mr create` 实际会因 GitLab 端无该 branch 而失败，但 SKILL.md 没有 fail-fast 检查。
- `pipeline` 在新旧 glab 子命令（`glab ci ...` vs `glab pipeline ...`）之间的 fallback 决策完全交给模型，无显式判定。
- `snow_dev_ai` username 在 `review-fixup` 与 README 里硬编码——bot 改名时需双处同步。
- `review-mr` 给 `glab mr view --comments` 没加 `--paginate`，超长评论 MR 可能漏抓评论。
- llmdoc 加载逻辑在 `review-mr` 与 `review-fixup` 内重复——索引格式变化要同步两处。
