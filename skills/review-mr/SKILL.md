---
name: glab:review-mr
description: Conduct a multi-dimensional code review of a GitLab MR (correctness, security, performance, readability, test coverage, project conventions). When llmdoc/ exists in cwd, loads relevant architecture / guides / reference docs as authoritative criteria. Use when user says review MR <N> / 评审 MR / 帮我看看这个 MR / code review.
allowed-tools: Bash, Read, Glob, Grep
---

# /glab:review-mr — 多维 MR Review（结合 llmdoc）

## Preconditions

依赖 `glab` CLI 已安装并完成 git.snowsse.cn 认证。

执行前先跑：`glab auth status`。失败则调用 `glab:setup` 输出教程，**终止本 skill**。

---

## 参数

- MR 编号（iid）：**必填**。如果用户没传：
  - 提示用户："/glab:review-mr 需要 MR 编号，例如 `/glab:review-mr 10`。如果想 review 当前分支 MR，可以先跑 `/glab:mr` 拿到编号"
  - 终止

---

## 主流程

### 1. 拉 MR 元数据 + diff

```bash
glab mr view <iid> --comments > /tmp/glab-mr-<iid>-view.txt
glab mr diff <iid> > /tmp/glab-mr-<iid>-diff.patch
```

用 Read 工具读这两个文件。从 view 输出提取：
- 标题 / 描述
- 源/目标分支
- 现有评论（含已有的 review 反馈）
- 流水线状态

### 2. 加载 llmdoc 相关上下文（动态、可选）

检测 cwd 下是否存在 llmdoc 入口：

```bash
([ -f llmdoc/startup.md ] || [ -f llmdoc/index.md ]) && echo "EXISTS" || echo "MISSING"
```

#### 若存在

**a. 优先按 llmdoc 启动顺序加载：**

- 如果存在 `llmdoc/startup.md`，按其中定义的启动阅读顺序加载文档。
- 如果没有 `llmdoc/startup.md`，读 `llmdoc/index.md`（若存在），用它做文档路由索引。

**b. 轻量 fallback：**

- 读 `llmdoc/must/*.md`（若存在），作为强约束。
- 读 `llmdoc/reference/coding-conventions.md`（若存在），作为 readability 维度依据。

**c. 定向加载：**

从 diff 里提取关键词信号：
- 改了哪些**文件路径**（目录名、包名、模块名、领域名）
- 改了哪些**符号名**（函数名、类名、常量）
- commit message / MR description 里的关键词

把关键词跟 llmdoc 索引里的文档路径和描述做匹配，最相关的 3-5 篇 llmdoc 文档 Read 进来：
- 优先读命中当前改动领域的 `architecture/`、`guides/`、`reference/` 文档。
- 如果候选文档过多，先读最像"总览 / must / architecture / convention"的文档。

#### 若不存在

跳过此步，review 只基于 diff + 通用知识。**在最终报告里明确标注**："本 review 未结合 llmdoc 项目规范（cwd 下未发现 `llmdoc/`）"。

### 3. 六维 review

按以下维度逐项评估，每个维度先写结论（无问题 / 有 N 个问题），再列具体 finding。

#### 维度 1：正确性（Correctness）

- 代码是否实现了 MR 标题/描述里说的目标？
- 边界条件（空数组、null、undefined、并发）有没有处理？
- 错误处理是否完整？
- 是否有逻辑 bug、off-by-one、错误的运算符？

#### 维度 2：安全性（Security）

- 用户输入有无未经校验直接进入 SQL/shell/HTML/正则？（OWASP Top 10）
- 敏感信息（token、PII、密码）有无泄露到日志、错误消息、URL？
- 权限边界有无绕过？（特别是 background ↔ content script 通信）
- 第三方依赖有无引入已知漏洞？

#### 维度 3：性能（Performance）

- 大循环 / O(N²) 是否合理？
- 是否有不必要的复制（深拷贝、JSON.parse(JSON.stringify(...))）？
- UI / DOM 操作有无引发布局抖动或不必要渲染？如存在相关 llmdoc，引用对应 llmdoc 文档。
- 内存泄漏（监听器未解绑、定时器未清理、大对象未释放）？

#### 维度 4：可读性（Readability）

- 命名是否清晰、表意？
- 函数长度是否合理（一个函数干一件事）？
- 注释是否在该有的地方有、不该有的地方没有（解释 why 而非 what）？
- 是否符合 llmdoc 相关文档中的 coding conventions（如存在）？

#### 维度 5：测试覆盖（Test Coverage）

- 这次改动有无对应的单元 / 集成测试？
- 缺哪些测试场景？（特别是 happy path 之外的）
- 已有测试是否更新（避免改了实现但测试还是老 mock）？

#### 维度 6：项目规范契合度（Convention Compliance）

只有在 llmdoc 存在时才评估。逐条对照：

- 已加载的 `llmdoc/must/*.md` 约束有无违反？
- 已加载的 `architecture/` 文档里描述的设计原则有无违反？
- 已加载的 `guides/` 文档里说的标准操作步骤是否被遵循？

每条 finding 引用 llmdoc 文档路径作为依据。

### 4. 输出报告

格式：

```markdown
# MR Review: !<iid> <title>

> 源分支: <source> → <target>
> Review 时间: <now>
> 项目规范来源: <llmdoc/ 已加载 X 篇 | 未加载，仅基于通用知识>

## 总体评价

<2-3 句话总结：可以合并 / 需要修复 N 个问题再合并 / 设计存疑需要讨论>

## 维度汇总

| 维度 | 结论 |
|---|---|
| 正确性 | ✓ 无问题 / ⚠ N 个问题 |
| 安全性 | ... |
| 性能 | ... |
| 可读性 | ... |
| 测试覆盖 | ... |
| 项目规范 | ... / N/A（无 llmdoc） |

## Findings

### 🔴 必须修复（Must Fix）

#### 1. <文件>:<行号> — <一句话标题>
**问题**: 详细说明
**依据**: <llmdoc 引用 / 通用最佳实践>
**建议**:
\`\`\`diff
- 原代码
+ 改后代码
\`\`\`

### 🟡 建议改进（Should Improve）

<同样格式>

### 🟢 加分项（Nice to Have）

<同样格式>

### 📝 风格分歧（Style Preference）

<可选；表格形式列出>

## 测试建议

<列出建议补充的测试场景>

## 结论

<是否建议 ready / approve / 仍需修改>
```

### 5. 重要约束

- ✅ 只产出 review 报告，**不直接改代码**
- ✅ 每条 finding 都要给文件路径 + 行号 + 修复建议（若可行的话给 diff 片段）
- ✅ 引用 llmdoc 文档时给完整相对路径
- ❌ 不在 GitLab 上自动发评论（让用户决定要不要发）
- ❌ 不修改 MR 状态（不 approve / 不 merge / 不 ready）
- ❌ 不要做"无脑列优点"的彩虹屁——只列**有意义的反馈**
