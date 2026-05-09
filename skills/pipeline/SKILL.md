---
name: glab:pipeline
description: Show the latest GitLab CI pipeline status for the current branch (or specified branch). Highlights failed jobs and provides commands to view their logs. Use when user asks pipeline / CI / 流水线 / 构建状态 / 跑得怎么样.
allowed-tools: Bash
---

# /glab:pipeline — 当前分支 CI 状态

## Preconditions

依赖 `glab` CLI 已安装并完成 git.snowsse.cn 认证。

执行前先跑：`glab auth status`。失败则调用 `glab:setup` 输出教程，**终止本 skill**。

---

## 参数

- 分支：默认当前分支；用户可指定其他分支（如 "看 develop 的 pipeline"）

---

## 主流程

### 1. 拿分支名

```bash
git rev-parse --abbrev-ref HEAD
```

如果用户指定了分支，覆盖。

### 2. 看 pipeline 状态

```bash
glab ci status --branch <分支>
```

或更详细：

```bash
glab ci view --branch <分支>
```

注：不同 glab 版本子命令略有差异。如果上面失败，回退到：

```bash
glab pipeline status --branch <分支>     # 旧版
glab pipeline view --branch <分支>
```

### 3. 中文总结输出

```
## 分支 <branch> 最新 Pipeline

- 状态: <success / failed / running / canceled / skipped>
- 触发: <commit-sha-short> "<commit-message-short>"
- 开始: <created_at>
- 耗时: <duration>

### Stages

| Stage | 状态 | Jobs |
|---|---|---|
| build | ✓ success | 3/3 |
| test | ✗ failed | 2/4（job1, job3 失败） |
| deploy | - skipped | - |

### 失败 Jobs（如有）

#### <job-name>
- 失败原因（从输出能看到的）: ...
- 查看完整日志: `glab ci trace <job-id>`
- 重跑: `glab ci retry <job-id>`
```

### 4. 操作建议

- 状态 `failed` → 列出 `glab ci trace <id>` 命令让用户复制
- 状态 `running` → 提示用户隔几分钟再 `/glab:pipeline` 看结果
- 状态 `success` → 简短"全绿"，如果有 MR 就提示"可以推进 MR ready"
- 状态 `canceled` / `skipped` → 提示原因（commit message 含 `[skip ci]` 等）

### 5. 不要做的事

- ❌ 不要自动跑 `glab ci retry`（影响构建队列，让用户决定）
- ❌ 不要拉完整 trace 日志（动辄数千行，浪费 context；只在用户明确要求时才拉）
