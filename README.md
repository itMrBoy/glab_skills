# glab — GitLab CLI 快捷 Skill 包

把日常 GitLab 操作（看 MR / 跑 CI / review / 处理 AI 反馈）封装成一组可复用 skill，**不用记 glab 命令**，同时提供 Claude Code plugin 配置与 Codex plugin 配置。

## 安装

重启 Claude Code 后，输入 `/` 即可看到 `/glab:setup`、`/glab:mr` 等命令。

### Claude Code（团队成员，从远程仓库拉取）

```powershell
# 从 GitHub
claude plugin marketplace add https://github.com/itMrBoy/glab_skills.git
claude plugin install glab@snow-glab-marketplace
/reload-plugins
```

### Codex CLI / Codex Plugin

仓库现在同时包含 Codex 侧配置：

- `.codex-plugin/plugin.json`：声明 `glab` plugin 及 `skills: "./skills/"`
- `.agents/plugins/marketplace.json`：用于描述本地 Codex marketplace 插件入口

最简单的方式：将以下提示词发送给Codex CLI，并根据提示授权即可：
```powershell
帮我安装这个skills：https://github.com/itMrBoy/glab_skills.git
```

如果你的 Codex 环境直接读取 repo 根下的 `.codex-plugin/plugin.json`，当前仓库已经具备最小可用的 plugin manifest。

如果你准备走本地 marketplace 方式，请先确认 `.agents/plugins/marketplace.json` 里的 `source.path` 与实际 plugin 根目录一致，再接入使用。

如果你的 Codex 环境还没走 plugin marketplace，也可以继续用最简单的方式：把 `glab_skills/skills/` 目录加入 Codex 的 skill 搜索路径。

> 注意：本仓库的 skill 语义可在 Claude Code 与 Codex 之间复用，但 `SKILL.md` 里仍有少量 Claude Code 风格的工具名（如 `Read`、`Glob`）。在 Codex 下通常需要由 agent 自行映射等价能力，因此它是**语义可移植**，不是协议层的"完全一致"。

### 升级

修改本地或拉取远端更新后：

```powershell
claude plugin marketplace update snow-glab-marketplace
```

重启 Claude Code 生效。

---

## 前置依赖

需要本机已安装并认证 [`glab`](https://gitlab.com/gitlab-org/cli)：

```powershell
glab --version            # 检查是否安装
glab auth status          # 检查认证状态
```

如果两条之一报错，**直接跑 `/glab:setup`** —— 它会检测并输出对应的中文安装/认证教程（不会自动改你的环境）。

---

## Skill 清单

| 命令 | 自然语言触发 | 用途 |
|---|---|---|
| `/glab:setup` | "装 glab"、"配置 GitLab CLI" | 检测 glab 状态，输出安装与鉴权教程 |
| `/glab:mr` | "看当前 MR"、"view current MR" | 查看当前分支对应的 MR |
| `/glab:mr-create [target]` | "创建 MR"、"提个 PR" | 从当前分支创建 MR（默认 develop，draft 状态） |
| `/glab:mr-diff [MR]` | "看 MR 改了啥"、"summarize MR diff" | MR 变更中文总结 |
| `/glab:pipeline` | "看流水线"、"CI 状态" | 当前分支最新 pipeline 状态 |
| `/glab:commits-ahead [base]` | "比 develop 多了哪些 commit" | 列本分支领先的 commits |
| `/glab:review-mr <MR>` | "review MR 10"、"评审 MR" | 多维 review（结合 llmdoc 项目知识） |
| `/glab:review-fixup <MR>` | "处理 AI review"、"snow_dev_ai 反馈" | 拉 AI 机器人评论 → 评估合理性 → 修复 plan |

---

## 设计要点

- **不自动改用户环境**：`/glab:setup` 只检测和输出教程，不跑 winget、不写 token
- **llmdoc 集成**：`review-mr` 和 `review-fixup` 自动检测 cwd 下 `llmdoc/`，按主题定向加载文档作为评判依据
- **AI review 筛选**：`review-fixup` 按 `notes[].author.username == "snow_dev_ai"` 过滤评论
- **跨工具复用**：核心逻辑都写在 `SKILL.md` 里；Claude Code 可直接按 plugin 使用，Codex 可通过 plugin 或 skill 路径接入

---

## License

Internal use within Snow team.
