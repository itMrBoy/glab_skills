# How to Release and Distribute

`glab` 通过 Claude Code plugin 机制分发，由两层 manifest 决定身份：`.claude-plugin/marketplace.json`（分发源）与 `.claude-plugin/plugin.json`（plugin 自身）。详见 `reference/plugin-manifest.md`。

本 guide 覆盖三种场景：本地迭代、首次发布到 GitHub、团队成员升级。

## 场景 A：本地迭代（开发者）

适用：自己在改 SKILL.md / manifest，想立刻在自己 Claude Code 里看到效果。

### 1. 首次添加（仅一次）

```powershell
claude plugin marketplace add C:\code\glab_skills
claude plugin install glab@snow-glab-marketplace
```

第一条把 `marketplace.json` 注册成本地分发源，第二条按 `marketplace.json#plugins[].source = "./"` 指向当前仓库根。

### 2. 改完代码后

```powershell
claude plugin marketplace update snow-glab-marketplace
```

然后**重启 Claude Code**（plugin 在冷启动加载，不重启不会生效）。

仅改 SKILL.md 内容时**不需要** bump 版本号——`update` 会重新读最新文件。

### 3. 改了 plugin.json 字段时

字段改动按下表处理：

| 改了什么 | 是否要 bump version |
|---|---|
| `description` / `author` 微调 | 否 |
| 新增 / 删除 / 重命名 skill | **是**（patch 或 minor） |
| 改 frontmatter `name` 或 `allowed-tools` | **是**（patch） |
| 破坏性改动（移除 skill / 改命令名） | **是**（minor 或 major） |

版本号语义（半正式）：
- `0.x.y` 阶段：minor 表示破坏性，patch 表示新增 skill 或不破坏的修复。
- 升到 `1.0.0` 后切回严格 SemVer。

## 场景 B：首次发布到 GitHub

仓库目前 `main` 分支**无任何 commit**（详见 `must/project-basics.md` 与 `memory/doc-gaps.md` G1）。把代码推上 `https://github.com/itMrBoy/glab_skills.git` 之前 README 中的 GitHub 安装路径不可用。

### 步骤

```powershell
git status                          # 确认所有文件 untracked

git add .claude-plugin/ skills/ README.md .gitignore llmdoc/
# .claude/ 已被 .gitignore 屏蔽，不入库

git commit -m "feat: initial release of glab plugin (8 skills + llmdoc)"

git push -u origin main
```

注意：
- `.claude/` 是本机运行时偏好（如 `settings.local.json`），永远不入库。
- 不要 add `.claude-plugin/` 之外、与分发无关的文件。
- 如果上游远端要求 PR 而非 direct push，先 `git push -u origin main:initial-release` 后开 PR。

### push 后回写 README

push 成功后**必须**移除 README 顶部的 G1 ⚠️ 提示段（位于 `# glab` 标题下方），否则误导读者认为还不能用 GitHub 路径。同步更新 `memory/doc-gaps.md` G1 条状态。

### 打 release tag（推荐，非必需）

```powershell
git tag -a v0.1.0 -m "Initial release"
git push origin v0.1.0
```

Claude Code plugin 当前不强依赖 git tag，但 tag 可以让团队 pin 版本（`marketplace add <git-url>@v0.1.0`，需验证当前版本 Claude Code CLI 是否支持该语法）。

## 场景 C：团队成员从 GitHub 拉取与升级

适用：你是使用方，不维护代码。

### 首次安装

```powershell
claude plugin marketplace add https://github.com/itMrBoy/glab_skills.git
claude plugin install glab@snow-glab-marketplace
```

或简写：

```powershell
claude plugin marketplace add itMrBoy/glab_skills
```

重启 Claude Code 后，输入 `/` 即可看到 `/glab:setup`、`/glab:mr` 等命令。

### 升级

```powershell
claude plugin marketplace update snow-glab-marketplace
```

然后重启 Claude Code。这条命令会从 GitHub 拉最新内容到本地缓存。

### 验证安装成功

跑 `/glab:setup`——它只读取 glab 状态并输出教程，是最安全的烟测。

## 场景 D：Codex CLI 用户

Codex CLI 不识别 plugin / marketplace metadata，但识别 `SKILL.md`。Codex 用户把 `glab_skills/skills/` 目录加进 Codex skill 搜索路径即可。

注意：`SKILL.md` 内多处用了 Claude Code 专属工具名（"用 Read 工具"、`Glob llmdoc/must/*.md`），Codex 下能否运行依赖 LLM 自行映射，**不是协议层兼容**（详见 `memory/doc-gaps.md` G2）。

## 已知失败点

- **改完没重启 Claude Code**：`marketplace update` 只刷新缓存，不热加载。一定要重启。
- **改了 frontmatter `name` 没改父目录名**：slash 命令发现失败，模型也找不到。三处（目录 / frontmatter / 用户调用）必须同步。
- **add 了路径却没 install**：`marketplace add` 只是注册分发源，还要 `plugin install glab@snow-glab-marketplace` 才实际生效。
- **本机有未提交改动时切到 GitHub 安装**：会拿到旧版本（远端无你本地的改动），现象是命令行为不一致。
- **GitHub 仓库私有但用 https 克隆**：`marketplace add <https url>` 在私有仓库下需要预先配 git credential helper，否则拉取失败。

## Related Docs

- `reference/plugin-manifest.md` — manifest 字段语义、命名映射。
- `must/project-basics.md` — 仓库当前状态。
- `memory/doc-gaps.md` — G1（GitHub 路径不可用）的最新状态。
- `guides/add-new-skill.md` — 新 skill 流程。
