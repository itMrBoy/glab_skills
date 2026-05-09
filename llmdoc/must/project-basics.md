# 项目基础

## 身份

`glab` 是以 Claude Code 为主、同时补充 Codex plugin 元数据的 skill 仓库。
- 归属团队：`snow_workspace_group`（marketplace.json#owner.name）。
- 作者：`liminghu <liminghu@snowtech.com.cn>`（plugin.json#author）。
- 默认 GitLab host：`git.snowsse.cn`（仅在 `setup` skill 显式声明，其它 skill 依赖 glab 默认 host）。
- 远端仓库：`https://github.com/itMrBoy/glab_skills.git`（git remote）。

## 双入口身份映射

| 层 | 字段 | 决定什么 |
|---|---|---|
| Claude Plugin | `.claude-plugin/plugin.json#name = "glab"` | slash 命令前缀 `/glab:`。 |
| Claude Marketplace | `.claude-plugin/marketplace.json#name = "snow-glab-marketplace"` | Claude 安装命令 `@snow-glab-marketplace`。 |
| Codex Plugin | `.codex-plugin/plugin.json#name = "glab"` | Codex 对插件的本地识别名；`skills: "./skills/"` 显式指向 skill 目录。 |
| Codex Marketplace | `.agents/plugins/marketplace.json#plugins[].source.path` | Codex 本地 marketplace 如何定位 plugin 根目录。 |
| Skill | `skills/<dir>/SKILL.md` frontmatter `name`（必须 = 父目录名） | Claude 下决定 `/glab:<name>` 后缀；Codex 下决定 skill 名称与路由语义。 |

## 8 个 Skill 索引

| Skill | 命令 | 一句话用途 |
|---|---|---|
| setup | `/glab:setup` | 诊断 glab 安装/认证并输出 Windows PowerShell 教程，**只读**。 |
| mr | `/glab:mr` | 看当前分支对应的 open MR，中文摘要。 |
| mr-create | `/glab:mr-create [target]` | 从当前分支创建 MR（默认 target=`develop`、默认 `--draft`）。 |
| mr-diff | `/glab:mr-diff [MR]` | 拉 MR diff 到 `/tmp` 后总结改动（按文件类别分组+风险点）。 |
| pipeline | `/glab:pipeline` | 看当前分支最近 CI pipeline 状态。 |
| commits-ahead | `/glab:commits-ahead [base]` | 列当前分支相对 base（默认 `develop`）的领先 commit 并查调试性提交。**唯一不依赖 glab 的 skill**。 |
| review-mr | `/glab:review-mr <MR>` | 6 维度多角度 review，结合 cwd 下 `llmdoc/` 项目知识。 |
| review-fixup | `/glab:review-fixup <MR>` | 拉 GitLab `snow_dev_ai` bot 的 AI review 评论，5 桶分类后输出修复 plan（不改代码）。 |

## 关键外部依赖

| CLI | 用于 | 必备性 |
|---|---|---|
| `glab` | 7/8 skill（除 `commits-ahead`） | 必备，未装/未认证时 skill 调 `/glab:setup` 终止。 |
| `git` | 5/8 skill | 必备（项目假定）。 |
| `jq` | 仅 `review-fixup` | 必备，缺失时提示 `winget install -e --id jqlang.jq` 终止。 |

目标平台：**Windows + PowerShell**（`setup` 教程仅给 Windows 分支）。但所有写临时文件的 skill 用 POSIX `/tmp/glab-mr-<iid>-*` 路径，**实际依赖 git-bash / WSL / Cygwin** 提供 `/tmp` 映射（README 未声明，见 `memory/doc-gaps.md`）。

## 仓库当前状态（2026-05-09 快照）

- `main` 已有 5 个本地 commit，`git log --oneline --decorate -1` 显示 `origin/main` 与本地 `HEAD` 对齐。
- 当前工作树待提交内容为：`README.md` 修改，以及新增的 `.codex-plugin/`、`.agents/`（Codex 接入相关元数据）。
- `.claude/` 被 `.gitignore` 屏蔽，仅本机运行时状态，不入库。
- GitHub 远端已不再是“空仓库”状态；README 中的 GitHub 安装路径是否包含最新改动，取决于下一次 push 是否完成。

## 没有的东西（避免误解）

- Claude Code 侧无 `commands/`、`agents/`、`hooks/`、`mcp_servers/`、`.mcp.json`。
- `.agents/plugins/marketplace.json` 是 Codex marketplace 元数据，不代表仓库里存在可执行 agent 逻辑。
- 无自动化测试 / 回归脚本 / lint 配置。
- 仓库自身的 `llmdoc/` 是给本 plugin 用的，与 review skill 期望"被评审项目"提供的 `llmdoc/` 是两件事。
