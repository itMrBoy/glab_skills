# 跨 Skill 工作约定

下列约束来自所有 SKILL.md 共有的"❌ 不要做"段落。改任何 skill 时这些是底线。

## 用户环境只读

- ❌ 不执行 `winget install` / `scoop install` / `choco install`。
- ❌ 不执行 `glab auth login`。
- ❌ 不写入或修改 token / GitLab 配置文件。
- ✅ 安装/认证类操作一律产出**中文教程文本**让用户手动执行（这是 `setup` skill 的全部职责）。

## Git 操作只读

- ❌ 不 `git push` / `git commit` / `git rebase` / 不 force push。
- ✅ 允许的 git 命令：`rev-parse`、`status --porcelain`、`log`、`diff --stat`、`fetch <ref>:<ref>`（仅 `commits-ahead` 在 base 缺失时用）。
- `mr-create` 默认 `--draft` 且会拒绝从主干分支（`main`/`develop`/`master`）创建 MR；执行前**强制人工确认**标题与描述。

## GitLab 远端只读（除 mr-create）

- 唯一会写远端状态的 skill 是 `mr-create`（创建 MR），且默认 draft + 必须用户确认。
- ❌ `pipeline` 不 auto retry。
- ❌ `review-fixup` 不 auto resolve discussion、不 auto comment、不 auto push。
- ❌ `review-mr` 不在 GitLab 加评论。

## 输出统一

- 全部 skill 输出**中文 Markdown**。
- `setup` 例外：教程文本逐字原样输出（不让模型改写）。
- 严重度 emoji 词汇统一：🔴 必须修复 / 🟡 建议改进 / 🟢 加分项 / ⚪ 风格分歧或误报 / 🔵 待决策 / 📝 风格表。

## 失败时降级

- **glab 未装/未认证**：依赖 glab 的 7 个 skill 在 Preconditions 段调 `glab auth status`，失败 → 调 `/glab:setup` 输出教程 → **终止本 skill**，不要绕过。
- **`commits-ahead` 例外**：纯本地 git，不调 `setup`。
- **`review-fixup` 多一道 jq 检查**：`jq --version` 失败 → 提示用户 `winget install -e --id jqlang.jq` → 终止。
- **llmdoc 缺失（仅 review skill）**：`[ -f llmdoc/index.md ]` 不存在时跳过加载，仍可运行；**报告头必须标注"未加载项目专属规范"**。不要静默降级。

## 临时文件约定

写到 `/tmp/glab-mr-<iid>-*`（POSIX 路径，依赖 bash shell）：
- `mr-diff`：`-diff.patch`。
- `review-mr`：`-view.txt` + `-diff.patch`。
- `review-fixup`：`-discussions.json` + `-diff.patch` + `-ai-findings.json`。

写完用 `Read` 工具按需加载，避免 stdout 把上下文撑爆。

## 工具命名（Claude Code 专属）

frontmatter `allowed-tools` 用 PascalCase：`Bash`、`Read`、`Glob`、`Grep`。Codex CLI 协议层不识别这些工具名，README 的"完全跨工具兼容"声明在严格意义上不成立——见 `memory/doc-gaps.md`。

## 用户参数读取

8 个 skill 都没有 `argument-hint` 字段，参数解析全靠模型从用户自然语言抽取。已知默认值：
- `mr-create [target]` → `develop`
- `commits-ahead [base]` → `develop`（说"上游"则用 `@{u}`）
- `mr-diff [MR]` → 当前分支推断
- `pipeline` → 当前分支
- `review-mr <MR>` / `review-fixup <MR>` → **必填**，缺失时提示用户先跑 `/glab:mr` 拿 iid。
