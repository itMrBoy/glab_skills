---
name: glab:commits-ahead
description: List commits the current branch has ahead of a base branch, with Chinese summary and warnings about debug-only commits (wip / fix typo / test). Use when user asks 比 develop 多了哪些 commit / commits ahead / 我加了哪些提交 / 这分支有哪些改动.
allowed-tools: Bash
---

# /glab:commits-ahead — 本分支领先 commits

## Preconditions

依赖 `git`（必装，本仓库假设已有）。**不依赖 glab**（纯本地 git 操作），所以无需调 `glab:setup`。

---

## 参数

- base 分支：默认 `develop`
  - 用户说 "比 main 多了哪些" → base = `main`
  - 用户说 "比上游多了哪些" → 用 `git rev-parse --abbrev-ref --symbolic-full-name @{u}` 拿 upstream
  - 没指定 → `develop`

---

## 主流程

### 1. 校验 base 分支存在

```bash
git rev-parse --verify <base> 2>&1
```

如果失败（base 不存在），先 `git fetch origin <base>:<base>` 拉一下，再次校验。仍失败就告诉用户"base 分支 `<base>` 不存在，请检查名字"，结束。

### 2. 拿 commits

```bash
git log <base>..HEAD --oneline --no-merges
```

得到形如：

```
abc1234 feat(logger): 接入 SnowLogger
def5678 fix(desensitization): 修复脱敏
...
```

### 3. 拿 diff stat 做总览

```bash
git diff <base>..HEAD --stat | tail -1
```

例如：`12 files changed, 234 insertions(+), 56 deletions(-)`

### 4. 中文总结输出

```
## <当前分支> 领先 <base> 的提交

总览: <N> 个 commit, <X> 文件改动 (+<additions> / -<deletions> 行)

### 提交列表

| SHA | 类型 | 描述 |
|---|---|---|
| abc1234 | feat | 接入 SnowLogger |
| def5678 | fix | 修复脱敏 |
| ... |

### 调试性提交检查（如有）

⚠ 检测到以下调试性提交，建议 `git rebase -i <base>` 合并或删除后再提 MR：

- xxx1234 "wip: 测试中"
- yyy5678 "fix typo"
```

#### 调试性提交识别规则

commit message 满足任一条件视为调试性：

- 含 `wip` / `WIP` / `fixup` / `squash`
- 含 `fix typo` / `修正错字` / `修复拼写`
- 含 `test` 但不是 conventional `test:` 前缀（即不以 `test:` / `test(` 开头）
- 字数 < 5（过短，可能是 `merge` / `update` 之类无意义提交）
- 含 `调试` / `临时` / `debug`

### 5. 结尾建议

如果**没有调试性提交**：提示 "可以 `/glab:mr-create` 提交 MR 了"。

如果**有调试性提交**：列 `git rebase -i <base>` 命令并简单解释（reword / squash / drop）。

### 6. 不要做的事

- ❌ 不要自动 rebase（破坏性，让用户决定）
- ❌ 不要 force push
