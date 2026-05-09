# glab — GitLab CLI 快捷 Skill 包

把日常 GitLab 操作（看 MR / 跑 CI / review / 处理 AI 反馈）封装成 Claude Code skill，**不用记 glab 命令**，跨 Claude Code 和 Codex CLI 通用。

> ⚠️ **当前状态**：本仓库尚未发布到 GitHub（`main` 分支无 commit）。请暂时使用下方 [Claude Code（本地路径）](#claude-code本地路径) 方式安装；[Claude Code（团队成员，从远程仓库拉取）](#claude-code团队成员从远程仓库拉取) 路径在首次 push 到 `https://github.com/itMrBoy/glab_skills.git` 之后才可用。详见 `llmdoc/memory/doc-gaps.md` 第 G1 条。

## 安装

### Claude Code（本地路径）

```powershell
claude plugin marketplace add C:\code\glab_skills
claude plugin install glab@snow-glab-marketplace
```

重启 Claude Code 后，输入 `/` 即可看到 `/glab:setup`、`/glab:mr` 等命令。

### Claude Code（团队成员，从远程仓库拉取）

```powershell
# 从 GitHub
claude plugin marketplace add https://github.com/itMrBoy/glab_skills.git
claude plugin install glab@snow-glab-marketplace

# 或 GitHub 简写
claude plugin marketplace add itMrBoy/glab_skills
claude plugin install glab@snow-glab-marketplace
```

### Codex CLI（未来适配）

Codex CLI 不识别 plugin/marketplace 元数据，但识别 SKILL.md。Codex 用户在配置里把 `glab_skills/skills/` 目录加进 skill 搜索路径即可，skill 内容**完全跨工具兼容**。

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
- **跨工具兼容**：所有逻辑写在 SKILL.md 里，Claude Code 和 Codex CLI 都能直接用

---

## License

Internal use within Snow team.
