# How to Release and Distribute

`glab` 当前同时维护 Claude Code 与 Codex 两套接入元数据：

- Claude：`.claude-plugin/marketplace.json` + `.claude-plugin/plugin.json`
- Codex：`.codex-plugin/plugin.json` + `.agents/plugins/marketplace.json`

详见 `reference/plugin-manifest.md`。

本 guide 覆盖四种场景：Claude 本地迭代、首次发布到 GitHub、团队成员升级、Codex 接入。

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

如果这是首次把仓库内容推上 `https://github.com/itMrBoy/glab_skills.git`，那么 push 完成前 README 中的 GitHub 安装路径对团队成员不可用。

### 步骤

```powershell
git status                          # 确认待发布文件状态

git add .claude-plugin/ .codex-plugin/ .agents/plugins/ skills/ README.md .gitignore llmdoc/
# .claude/ 已被 .gitignore 屏蔽，不入库；Codex 元数据是否一并入库，取决于你是否准备同时维护 Codex 入口

git commit -m "feat: initial release of glab plugin (8 skills + llmdoc)"

git push -u origin main
```

注意：
- `.claude/` 是本机运行时偏好（如 `settings.local.json`），永远不入库。
- `.codex-plugin/`、`.agents/plugins/` 只有在你确认要维护 Codex 入口时才入库；若入库，需同步验证路径与版本号。
- 如果上游远端要求 PR 而非 direct push，先 `git push -u origin main:initial-release` 后开 PR。

### push 后回写 README / doc-gaps

如果 README 顶部仍保留“尚未 push 时请优先本地安装”之类的阶段性提示，push 成功后应回写为更准确的条件化表述，并同步更新 `memory/doc-gaps.md` G1 条状态。

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

## 场景 D：Codex 用户 / Plugin

Codex 侧当前有三种接法：

### 1. 直接 plugin 方式

依赖仓库根下的 `.codex-plugin/plugin.json`：

```json
{
  "name": "glab",
  "version": "1.0.0",
  "skills": "./skills/"
}
```

适用：Codex 能直接把当前仓库根识别为 plugin 根目录。

### 2. 本地 marketplace 方式

依赖 `.agents/plugins/marketplace.json`。接入前先确认：

- `plugins[].name` 与 `.codex-plugin/plugin.json#name` 一致
- `plugins[].source.path` 指向**真实存在**的 plugin 根目录

当前仓库里的 marketplace 文件把 `source.path` 写成 `./plugins/glab`；如果仓库仍采用“repo 根即 plugin 根”的布局，这个路径需要先回写或调整目录结构。

### 3. skill-path 兜底方式

如果你的 Codex 环境还没接 plugin / marketplace，直接把 `glab_skills/skills/` 目录加入 skill 搜索路径即可。

注意：`SKILL.md` 内多处用了 Claude Code 风格工具名（如 `Read`、`Glob`），Codex 下通常要由 agent 映射为等价能力，因此它是**语义可移植**，不是协议层完全一致（详见 `memory/doc-gaps.md` G2）。

## 已知失败点

- **改完没重启 Claude Code**：`marketplace update` 只刷新缓存，不热加载。一定要重启。
- **改了 frontmatter `name` 没改父目录名**：slash 命令发现失败，模型也找不到。三处（目录 / frontmatter / 用户调用）必须同步。
- **add 了路径却没 install**：`marketplace add` 只是注册分发源，还要 `plugin install glab@snow-glab-marketplace` 才实际生效。
- **本机有未提交改动时切到 GitHub 安装**：会拿到旧版本（远端无你本地的改动），现象是命令行为不一致。
- **GitHub 仓库私有但用 https 克隆**：`marketplace add <https url>` 在私有仓库下需要预先配 git credential helper，否则拉取失败。
- **Codex marketplace 路径没对齐**：`.agents/plugins/marketplace.json#plugins[].source.path` 若指向不存在目录，Codex 无法通过 marketplace 找到 plugin 根。

## Related Docs

- `reference/plugin-manifest.md` — manifest 字段语义、命名映射。
- `must/project-basics.md` — 仓库当前状态。
- `memory/doc-gaps.md` — G1（GitHub 路径不可用）的最新状态。
- `guides/add-new-skill.md` — 新 skill 流程。
