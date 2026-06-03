# Reflection — review-mr / review-fixup 的 llmdoc 加载策略重构

## 背景

本次变更是对 `glab` plugin 的 review skill（`review-mr` 和 `review-fixup`）的 llmdoc 加载策略进行了重构：

- 移除了 SKILL.md 中硬编码的 keyword→doc 路由表（如 `logger`→`snow-logger-guide.md`、`脱敏`→`data-desensitization-architecture.md` 等），这些路由表属于被评审项目 `sws-browser-extension` 的专属知识，不应出现在通用 skill 定义中。
- 将 llmdoc 检测入口从只检查 `index.md` 扩展为 `startup.md || index.md`。
- 加载策略从硬编码的 4 步/8 行路由表改为通用匹配原则（startup 优先 → fallback → topic-targeted）。
- 定向匹配改为基于文件路径、符号名、评论 body 关键词的通用匹配。
- 统一降级标注术语为"llmdoc 项目规范"。
- `review-fixup` 额外将 MR iid 从必填改为可选（自动推导当前分支 open MR），并新增获取 `web_url` 用于报告展示。
- 同步更新了 `review-pipeline.md`、`working-agreement.md`、`project-overview.md` 中的加载策略描述和术语。

## 这次暴露出的问题

### 1. SKILL.md 中硬编码项目专属 keyword→doc 路由表

`review-fixup` 的 SKILL.md 之前内置了 8 行具体的路由映射（如 `logger` 对应 `snow-logger-guide.md`、`content script` 对应 `content-script-architecture.md` 等）。这些映射完全属于某个被评审项目（`sws-browser-extension`）的内部知识，与 skill 的通用定位矛盾。

**问题本质**：skill 应该是 llmdoc 的**消费者**而非**写入者**。具体项目的 llmdoc 结构应由项目自己定义（通过 `startup.md` 或 `index.md` 的启动顺序和文档索引），skill 只负责按通用原则消费。

### 2. llmdoc 加载策略在两个 SKILL.md 中复制粘贴

review-mr 和 review-fixup 的 llmdoc 加载逻辑（检测入口 → 启动顺序 → fallback → topic-targeted）在变更前几乎是逐字复制。这意味着：

- 任何加载策略的微调都需要同步改两处。
- 两处容易逐渐 diverge（事实上变更前 review-fixup 有 8 行路由表而 review-mr 没有，就已经 diverge 了）。

### 3. 术语不统一

变更前，不同文档对同一概念使用了不同说法：

| 文档 | 术语 |
|---|---|
| `review-mr/SKILL.md` | "本 review 未结合 llmdoc 项目规范" |
| `review-fixup/SKILL.md` | "未加载 llmdoc 项目规范" |
| `review-pipeline.md` | "llmdoc 规范来源" |
| `working-agreement.md` | "llmdoc 相关文档缺失" |
| `project-overview.md` | "未加载 llmdoc 项目规范" |

同一降级标注在 5 个地方出现了 3 种不同表述，会导致模型输出不一致。

### 4. review-fixup 的 MR iid 之前是必填

用户反馈：review-mr 和 review-fixup 都要求手动传入 MR iid，但 review-fixup 的使用场景通常是"处理当前分支的 AI review 评论"，当前分支往往只有一个 open MR，完全可以通过 `glab mr list --source-branch` 自动推导。review-mr 保持必填（主动评审任意 MR 的场景更常见），review-fixup 改为可选更合理。

## 这次应该被提升为稳定知识的内容

### 1. Skill 不内置具体项目的 keyword→doc 路由表

- SKILL.md 中**禁止**出现类似 `"logger" -> "snow-logger-guide.md"` 的硬编码映射。
- 定向加载应使用通用匹配原则：文件路径中的模块名/目录名/包名 → 评论 body 中的业务词/技术词/接口名 → llmdoc 索引中的文档路径和描述。
- 被评审项目若需要精准路由，应在自身的 `llmdoc/startup.md` 或 `llmdoc/index.md` 中定义启动阅读顺序和文档索引。

### 2. llmdoc 加载入口应支持 startup.md 优先于 index.md

- 检测顺序：`llmdoc/startup.md` → `llmdoc/index.md`。
- `startup.md` 存在时按其中定义的启动阅读顺序加载；不存在时才 fallback 到 `index.md` + `must/*.md` + `reference/coding-conventions.md`。
- 这一顺序应在所有涉及 llmdoc 加载的 skill 中保持一致。

### 3. 涉及 llmdoc 的术语应在所有文档中统一

- 降级标注统一为：**"未加载 llmdoc 项目规范"**（报告头）和 **"llmdoc/ 已加载 X 篇"**（已加载时）。
- 消费契约中的描述统一为：**"被评审项目的 llmdoc 期望结构"**。
- 任何涉及 llmdoc 的术语变更，必须全局搜索所有 SKILL.md 和 `llmdoc/` 文档确保一致。

## 下次遇到类似任务的操作提醒

1. **改 SKILL.md 的 llmdoc 加载逻辑时，同步检查 `architecture/review-pipeline.md`**
   - review-pipeline.md 是双路管线的"稳定定义源"，SKILL.md 是它的"执行层实现"。
   - 两者必须保持一致，否则模型读 pipeline 文档和读 SKILL.md 会得到不同指令。

2. **改术语时，全局搜索确保所有文档一致**
   - 搜索范围：`skills/*/SKILL.md` + `llmdoc/**/*.md`。
   - 特别关注：降级标注、llmdoc 加载状态描述、消费契约措辞。

3. **改 review-mr 时检查 review-fixup 是否需要同步**
   - 它们是**独立兄弟 skill**，不互相调用，但共享多套基础设施（llmdoc 加载、数据获取链、输出格式、安全边界）。
   - 任何共享基础设施的变更（如加载策略、临时文件命名、输出模板结构）都应同时检查两个 skill。

4. **新增"可选参数 + 自动推导"功能时，检查 working-agreement.md 的参数默认值表**
   - working-agreement.md 的"用户参数读取"段列出了所有 skill 的已知默认值，新增自动推导逻辑后应同步更新该表。
