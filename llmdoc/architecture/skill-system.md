# Skill System Architecture

8 个 skill 共享一组约定 + 一张推荐图。理解共享模式后，单个 SKILL.md 只需关注它独有的逻辑。

## 角色分组

| 组 | Skills | 共同点 |
|---|---|---|
| 基础查询 | `mr`, `mr-diff`, `pipeline`, `commits-ahead` | 只读，输出中文摘要。 |
| 状态写入 | `mr-create` | 唯一写远端（创建 MR），强制人工确认 + 默认 draft。 |
| 环境引导 | `setup` | 只读探测 + 教程文本，**所有其它 glab 依赖 skill 的 fallback**。 |
| AI Review | `review-mr`, `review-fixup` | 详见 `review-pipeline.md`。 |

## Cross-Skill 推荐图

```
                  ┌─────────────────────────────┐
                  │           setup             │
                  │  (auth/install fallback)    │
                  └─────────────┬───────────────┘
                  调用兜底（4 个）│
        ┌─────────┬──────────────┼──────────────┬──────────┐
        ▼         ▼              ▼              ▼          ▼
       mr     mr-create       mr-diff        pipeline   review-mr
        │     ↕(互相回指)         ▲                       review-fixup
        │                         │
        ├──→ mr-create（无 MR 时）│
        ├──→ mr-diff（draft 时）──┘
        ├──→ pipeline（CI 失败时）
        └──→ review-fixup（有未解决讨论时）

   commits-ahead ──→ mr-create（无调试性提交时推荐）
```

证据：
- `setup` 兜底：`skills/mr/SKILL.md`（Preconditions）、`skills/mr-create/SKILL.md`、`skills/mr-diff/SKILL.md`、`skills/pipeline/SKILL.md`、`skills/review-mr/SKILL.md`、`skills/review-fixup/SKILL.md` 都写"glab auth status 失败 → 调 `glab:setup` 输出教程并终止"。
- `mr` 推荐其它 skill：`skills/mr/SKILL.md`（操作建议段，按 MR 状态推荐 mr-diff/pipeline/review-fixup/mr-create）。
- `mr-create` ↔ `mr` 互相回指：`mr-create` 发现已有 MR 时回指 `/glab:mr`；`mr` 没找到 MR 时建议 `/glab:mr-create`。
- `commits-ahead` → `mr-create`：`skills/commits-ahead/SKILL.md`（无调试性提交分支）。
- `commits-ahead` 是**唯一不依赖 glab 也不调 setup** 的 skill。

## 共享约定

### Frontmatter 三件套

8 个 SKILL.md 全部使用且仅使用：
```yaml
name: <skill-name>           # 必须 = 父目录名
description: <一句话用途. Use when user says <中英自然语言触发短语>.>
allowed-tools: Bash[, Read[, Glob[, Grep]]]
```

未使用：`tools` / `namespace` / `argument-hint` / `model` / `disable-model-invocation`。`/glab:<name>` 命令名**不写在 SKILL.md 内**——由 plugin name + 目录名自动拼接（详见 `reference/plugin-manifest.md`）。

`allowed-tools` 分布：
- `Bash`：`setup`, `mr`, `mr-create`, `pipeline`, `commits-ahead`。
- `Bash, Read`：`mr-diff`（要读 `/tmp` patch）。
- `Bash, Read, Glob, Grep`：`review-mr`, `review-fixup`（要遍历 `llmdoc/` + 搜被评审代码）。

### Preconditions 段模板（4 个 skill 共用）

`mr` / `mr-create` / `mr-diff` / `pipeline` 都按这个模板写：

```
1. 跑 `glab auth status`
2. 失败 → 调 `glab:setup` 输出教程
3. 终止本 skill，不要继续
```

`review-mr` 同模板。`review-fixup` 跑 `glab auth status && jq --version`（失败分别提示 setup 或 `winget install -e --id jqlang.jq` 后终止）。`commits-ahead` 不调 glab、无此模板。

### 临时文件命名

`/tmp/glab-mr-<iid>-<role>.<ext>`（POSIX 路径，依赖 bash shell）：

| Skill | 文件 |
|---|---|
| `mr-diff` | `-diff.patch` |
| `review-mr` | `-view.txt`、`-diff.patch` |
| `review-fixup` | `-discussions.json`、`-diff.patch`、`-ai-findings.json` |

写完后用 `Read` 工具按需加载，避免 stdout 灌满上下文。

### 严重度 emoji 词汇

- 🔴 必须修复 (Must Fix / valid bug)
- 🟡 建议改进 (Should Improve / valid improvement)
- 🟢 加分项 (Nice to Have)
- ⚪ 风格分歧 / 误报 (Style Preference / False Positive)
- 🔵 待决策 (Uncertain — 缺信息)
- 📝 风格分歧表

`review-mr` 用前 4 + 📝；`review-fixup` 用 🔴/🟡/⚪/🔵。

### 中文 Markdown 输出

所有 skill 输出中文 Markdown。`setup` 例外：教程文本逐字原样输出（不让模型改写）。

## 不变量与失败模式

| 不变量 | 失败模式 / 防御 |
|---|---|
| skill 名 == 父目录名 == frontmatter `name` | 三者不一致会让命令发现失效；改名时三处必须同步。 |
| 用户环境只读 | 见 `must/working-agreement.md`。`setup` 自身列出黑名单（不 winget/scoop/choco/auth login/写 token）。 |
| `glab auth status` 是 7 个 skill 的统一前置 | **文字约定，无机制保障**——靠模型遵守 SKILL.md 步骤；无回归测试。 |
| `mr-create` 不从主干分支创建 MR | HEAD 是 `main`/`develop`/`master` → 拒绝并结束。 |
| `mr-create` source branch 必须已 push | **未防御**——`glab mr create` 实际可能因远端无该分支失败，SKILL 未提示（见 `memory/doc-gaps.md`）。 |
| detached HEAD（`git rev-parse --abbrev-ref HEAD` = `HEAD`） | `mr` 直接结束并提示。 |
| `glab mr list --source-branch <分支>` 返回空 | `mr` 提示用 `/glab:mr-create`；`mr-diff` 让用户明确指定 iid。 |
| `mr-diff` 多条 MR 命中 | 列出让用户挑。 |
| `commits-ahead` 的 base 不存在 | 先尝试 `git fetch origin <base>:<base>`，仍失败则告知用户结束。 |
| `pipeline` glab 子命令多版本 | 主路径 `glab ci ...`，旧版回退 `glab pipeline ...`，**判定逻辑交给模型**（无显式 fallback 代码）。 |

## 命令具体行为速查

| Skill | 主命令链 | 默认值 / 关键参数 |
|---|---|---|
| `setup` | `glab --version`, `glab auth status --hostname git.snowsse.cn` | host 硬编码 `git.snowsse.cn`。 |
| `mr` | `glab mr list --source-branch <分支> --output json` → `glab mr view <iid>` | 当前分支由 `git rev-parse --abbrev-ref HEAD` 取。 |
| `mr-create` | `glab mr list --output json` → `git log/diff <target>..HEAD` → `glab mr create --target-branch <t> --title ... --description ... --draft` | target 默认 `develop`；默认 `--draft`；强制人工确认。 |
| `mr-diff` | `glab mr diff <iid> > /tmp/...patch` → `glab mr view <iid>` | 文件按 5 类（测试/配置/文档/样式/源码）分组，diff>1000 行时每组前 5 + 省略；总结 ≤500 字。 |
| `pipeline` | `glab ci status --branch <分支>`，旧版 `glab pipeline ...` 回退 | 默认当前分支；禁止 auto retry 与拉完整 trace。 |
| `commits-ahead` | `git rev-parse --verify <base>` → `git log <base>..HEAD --oneline --no-merges` → `git diff <base>..HEAD --stat` | base 默认 `develop`，"上游"用 `@{u}`；不调 glab；调试性提交关键词：wip/fixup/squash/fix typo/test/调试/临时/debug/字数<5。 |
| `review-mr` | 见 `review-pipeline.md` | iid 必填。 |
| `review-fixup` | 见 `review-pipeline.md` | iid 可选（无 iid 时自动推导当前分支 open MR）。 |

## 已知耦合点

- `setup` 用 `--hostname git.snowsse.cn`，其它 skill 的 `glab auth status` **不带 hostname**——多 host 环境下前置检查可能通过但 MR 命令失败。详见 `memory/doc-gaps.md`。
- `review-mr` 与 `review-fixup` 的 llmdoc 加载逻辑近乎复制粘贴——以后 llmdoc 索引格式变化需同步两处。详见 `review-pipeline.md`。
- `/tmp/glab-mr-<iid>-*` 约定散落在 3 个 SKILL.md，未抽到顶层文档；本文件是它的唯一记录处。

## Related Docs

- `architecture/review-pipeline.md` — review-mr / review-fixup 深度。
- `reference/plugin-manifest.md` — frontmatter / 命令命名映射。
- `must/working-agreement.md` — 强约束。
- 各 SKILL.md：`skills/<name>/SKILL.md`。
