# Plugin Manifest 与命名映射

三个 manifest 文件 + 命令名生成规则的稳定查表。

## `.claude-plugin/plugin.json`

描述"一个 plugin 自己是什么"。本仓库写法：

```json
{
  "name": "glab",
  "version": "0.1.0",
  "description": "...",
  "author": "blank3 <liminghu@snowtech.com.cn>"
}
```

| 字段 | 决定什么 | 必备 |
|---|---|---|
| `name` | **slash 命令前缀**（`/<name>:<skill>`），plugin 唯一标识 | 是 |
| `version` | semver 版本号 | 是 |
| `description` | 一句话描述 | 是 |
| `author` | 归属（`Name <email>` 形式） | 是 |

**未使用但合法**：`commands` / `skills` / `agents` / `hooks` / `mcpServers`——Claude Code plugin 用约定优于配置，component 通过子目录的存在自动发现，不需在 plugin.json 列举。

## `.claude-plugin/marketplace.json`

描述"一个分发源对外暴露哪些 plugin"。本仓库写法：

```json
{
  "name": "snow-glab-marketplace",
  "owner": { "name": "snow_workspace_group" },
  "description": "...",
  "plugins": [
    { "name": "glab", "source": "./", "description": "..." }
  ]
}
```

| 字段 | 决定什么 | 必备 |
|---|---|---|
| `name` | 安装命令 `<plugin>@<marketplace>` 的后半 | 是 |
| `owner.name` | marketplace 归属（团队） | 是 |
| `plugins[].name` | **必须等于** 目标 plugin 的 `plugin.json#name`，否则 install 找不到 | 是 |
| `plugins[].source` | plugin 根相对 marketplace.json 的位置；`"./"` = 同根（单 plugin 仓库） | 是 |
| `plugins[].description` | marketplace 列表展示文案 | 是 |

多 plugin 仓库写法：每条 `source` 指不同子目录（如 `plugins/foo/`），每个子目录里再放各自 `.claude-plugin/plugin.json`。

## SKILL.md frontmatter

每个 `skills/<name>/SKILL.md` 的开头 YAML 块。本仓库统一只用 3 字段：

```yaml
---
name: <skill-name>
description: <一句话用途. Use when user says <中英自然语言触发短语>.>
allowed-tools: Bash[, Read[, Glob[, Grep]]]
---
```

| 字段 | 规则 |
|---|---|
| `name` | **必须等于父目录名**。三处一致：目录名 / frontmatter `name` / 启动后的命令后缀。 |
| `description` | Anthropic 标准必填字段；末尾用自然语言"Use when user says ..."列触发短语，靠模型匹配。 |
| `allowed-tools` | Anthropic 标准字段名，但本仓库用的 `Bash`/`Read`/`Glob`/`Grep` 是 **Claude Code 扩展 PascalCase 工具名**（与 Anthropic 文档示例的 `computer_20250124` 等 ID 风格不同）。 |

**未使用**：`tools` / `namespace` / `argument-hint` / `model` / `disable-model-invocation`。参数解析全靠模型从用户自然语言抽取，无 schema。

## 命令前缀映射规则

`/glab:mr-create` 是怎么来的？两段拼接：

```
/<plugin.json#name>:<skill-dir-name>
└── plugin.json#name = "glab"   (.claude-plugin/plugin.json)
                       └─ skills/mr-create/   (目录名)
                                  └─ SKILL.md frontmatter name: mr-create  (必须一致)
```

8 个 skill 的全部映射：

| skill 目录 | frontmatter `name` | 命令 |
|---|---|---|
| `skills/setup/` | `setup` | `/glab:setup` |
| `skills/mr/` | `mr` | `/glab:mr` |
| `skills/mr-create/` | `mr-create` | `/glab:mr-create` |
| `skills/mr-diff/` | `mr-diff` | `/glab:mr-diff` |
| `skills/pipeline/` | `pipeline` | `/glab:pipeline` |
| `skills/commits-ahead/` | `commits-ahead` | `/glab:commits-ahead` |
| `skills/review-mr/` | `review-mr` | `/glab:review-mr` |
| `skills/review-fixup/` | `review-fixup` | `/glab:review-fixup` |

**SKILL.md 内不写 `/glab:<name>` 命令名**——名字不是它的字段，是运行时拼接出来的。

## 仓库布局：`.claude-plugin/` vs `.claude/`

| 目录 | 用途 | 是否分发 |
|---|---|---|
| `.claude-plugin/`（dash） | plugin manifest（plugin.json + marketplace.json）= **分发产物** | ✅ 进库 |
| `.claude/`（dot） | Claude Code 本机运行时状态（settings.local.json、cache 等） | ❌ 被 `.gitignore` 屏蔽 |

两个名字仅 1 个字符之差但语义完全不同。本仓库 `.claude/settings.local.json` 仅含 `permissions.allow: ["Skill(llmdoc:llmdoc)"]`——本机用户的 skill 调用授权，不入库。

## 触发短语机制

slash 命令名 `/glab:<name>` **不写在 SKILL.md 内**。skill 的所有触发都靠 frontmatter `description` 末尾的自然语言句子驱动，例如：

```
description: View the GitLab merge request associated with the current git branch. Use when user says 看当前 MR / 当前分支 MR / view current MR / show me the merge request / 这个分支有 MR 吗.
```

模型读到用户消息后匹配这些短语决定调用哪个 skill。这是约定优于配置的延伸——也意味着触发短语是 SKILL.md 里"可调"的部分，要在改动时谨慎（与 README 第 59-68 行的清单同步）。

## 安装命令字段对应

```
claude plugin marketplace add <path-or-url>     # <path-or-url> 必须含 .claude-plugin/marketplace.json
claude plugin install glab@snow-glab-marketplace
                      │     └── marketplace.json#name
                      └────── marketplace.json#plugins[].name == plugin.json#name
claude plugin marketplace update snow-glab-marketplace   # 按 marketplace 名刷新
```

`marketplace add` 的 3 种入参（README 列出）：本地绝对路径（开发）/ git HTTPS URL（分发）/ GitHub `owner/repo` 简写（仅 GitHub）。

`marketplace update` vs `plugin install`：前者刷新 marketplace 索引（rescan / git pull），后者首次安装到本地 plugin 缓存。两者都需要重启 Claude Code 才生效。`marketplace update` 是否自动升级已安装 plugin **README 未明说**（见 `memory/doc-gaps.md`）。

## Related

- `architecture/skill-system.md` — frontmatter 三件套在共享约定中的位置。
- `README.md` — 官方安装/升级流程。
- 上游官方文档：https:/code.claude.com/docs/en/plugins-reference（仓库内无本地副本）。
