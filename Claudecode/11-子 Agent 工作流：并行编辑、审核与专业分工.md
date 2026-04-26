# 子 Agent 工作流：并行编辑、审核与专业分工

子 Agent 的核心价值是把复杂任务拆成可控的专业角色：有人只读分析，有人实现指定文件，有人做安全审核，有人验证测试。要用好这个机制，最重要的是把边界说清楚——文件所有权、写入范围、审核顺序、冲突怎么处理。

⚠️ 子 Agent 能加快操作速度，也会放大出错的速度。涉及删文件、大范围重构、改权限、装依赖、提交代码、联网、碰密钥的任务，都应先做只读分析和计划确认，再进入实现阶段。

## 使用场景

**适合用子 Agent 的情况：**

- 大型代码库探索：让只读 Agent 搜索架构、调用链、测试入口，避免主对话被大量日志塞满。
- 并行编辑：把不同文件、不同模块分配给不同 Agent，并声明"只允许修改自己负责的文件"。
- 专业分工：前端 Agent、后端 Agent、测试 Agent、安全审核 Agent、文档 Agent 各司其职。
- 审核与复核：实现 Agent 完成后，由另一个只读 Agent 做代码审查、风险扫描和测试建议。
- 长任务隔离：让子 Agent 处理日志分析、依赖排查、文档整理等噪声大的任务，主对话只收结论。
- 多人协作仓库：多个 Agent 同时操作同一仓库时，用明确的文件清单避免互相覆盖。

**不适合用子 Agent 的情况：**

- 单文件小修，不需要额外的上下文隔离。
- 需求还没澄清，多个 Agent 会把歧义扩散出去。
- 需要严格人工审批的高风险操作——比如删目录、改权限、批量重构、提交或回退代码。

## 子 Agent 与 Agent Teams 的区别

两者都能把工作放到独立上下文，但适用场景不同：

- **子 Agent**：在独立上下文中工作，把结果摘要返回主对话，保护主对话不被大量搜索日志或文件内容污染。可以配置工具白名单/黑名单、模型（比如用 Haiku 省成本）、权限模式、hooks、持久记忆、fork 和 worktree 隔离。
- **Agent teams**：适合多个 Claude Code 实例需要互相沟通、共享任务列表、协同推进的复杂工作。是进阶实验功能，应谨慎使用。
- 并行写代码时，无论用哪种方式，都要让每个参与者有清晰的文件所有权。更复杂的并行场景应考虑独立 git worktree 或非常严格的文件清单，避免两个 Agent 改同一个文件。

## 创建自定义子 Agent

自定义子 Agent 存放在 `.claude/agents/`（项目级）或 `~/.claude/agents/`（用户级），每个 Agent 是一个 Markdown 文件，包含 frontmatter 配置和系统提示词。

**标准文件格式：**

```markdown
---
name: security-reviewer
description: 审查代码安全风险。在代码修改后自动调用，或用户说"帮我做安全审查"时调用。
tools: Read, Grep, Glob, Bash
model: sonnet
---
你是一名资深安全工程师。只读审查代码，重点检查：
- 注入漏洞（SQL、XSS、命令注入）
- 认证和授权缺陷
- 代码中的密钥或凭据
- 不安全的数据处理

给出具体行号和修复建议。不要修改文件。
```

**frontmatter 核心字段：**

| 字段 | 说明 |
|------|------|
| `name` | Agent 名称（用于显示） |
| `description` | 触发时机说明，Claude 根据此决定何时自动委派 |
| `tools` | 允许使用的工具列表 |
| `model` | 指定模型，如 `haiku`（省成本）、`sonnet`、`opus` |
| `permissionMode` | 权限模式：`default`、`acceptEdits`、`plan`、`auto` |
| `isolation` | `worktree`：在独立 git worktree 中运行，彻底隔离 |

**用 `/agents` 命令管理：** 在会话内输入 `/agents` 打开交互式界面，可以查看内置和自定义 Agent、创建新 Agent、编辑配置、删除 Agent。

也可以通过 CLI 临时定义：

普通终端命令：

```bash
claude --agents '{"reviewer":{"description":"只读审查代码","prompt":"你是代码审查员，只读不改","tools":["Read","Grep"],"model":"sonnet"}}'
```

**推荐工作流：**

1. 主对话先定义目标、风险和写入范围。
2. 只读 Agent 收集上下文，不修改文件。
3. 主对话把任务拆分到具体文件或模块。
4. 实现 Agent 只改自己负责的文件。
5. 审核 Agent 只读检查改动、测试缺口和风险。
6. 主对话统一决定是否测试、提交或继续修改。

## 推荐提示词模板

Claude Code 对话输入：

```markdown
你是子 Agent A，只负责只读分析。请检查当前仓库中与用户登录流程相关的文件、测试入口和潜在风险。

约束：
- 不要修改任何文件。
- 不要安装依赖。
- 不要联网。
- 不要运行会写入磁盘或修改数据库的命令。
- 输出只包含：相关文件、调用链、测试建议、风险点。
```

Claude Code 对话输入：

```markdown
你是子 Agent B，只负责实现以下文件：
- src/auth/login.ts
- tests/auth/login.test.ts

写入边界：
- 只允许修改上面两个文件。
- 不要回退、删除或格式化其他文件。
- 如果发现必须动其他文件，先停下来说明原因。

任务：
根据主对话中的计划修复登录失败时错误码不一致的问题，并补充测试。
高风险动作禁止：删除文件、批量重构、改权限、安装依赖、提交代码、联网、处理密钥。
```

Claude Code 对话输入：

```markdown
你是审核 Agent。请只读审查本轮改动。

重点检查：
- 是否修改了声明范围外的文件。
- 是否引入安全风险或密钥泄露。
- 是否存在未覆盖的错误路径。
- 是否需要补充测试。
- 是否有危险命令、联网、依赖安装或提交代码的痕迹。

不要修改文件。请按严重程度列出发现。
```

普通终端命令：

```bash
git status --short
git diff -- src/auth/login.ts tests/auth/login.test.ts
```

用途：在让审核 Agent 工作前，先确认本轮改动范围是否符合文件所有权。

## 错误示范

Claude Code 对话输入：

```markdown
开几个子 Agent，把这个项目所有问题都修好，能改哪里就改哪里，最后直接提交。
```

**问题在哪：**

- 没有文件所有权，多个 Agent 容易互相覆盖。
- "所有问题"范围过大，没法验收。
- "能改哪里就改哪里"会带来大量无关重构。
- "直接提交"在没看 diff 和跑测试之前不能做。

普通终端命令：

```bash
claude -p "重构整个项目并删除没用的文件"
```

**问题在哪：** 没有只读分析，没有写入范围，删文件是高风险操作，非交互模式下也没有审查机会。

Claude Code 对话输入：

```markdown
你负责文档，顺便把看到的代码风格问题也修一下。
```

**问题在哪：** "顺便"破坏了任务边界。文档 Agent 一旦开始改代码，其他正在改代码的 Agent 就可能产生冲突。

## 正确示范

Claude Code 对话输入：

```markdown
我们使用多 Agent 工作流完成本次改动。

全局规则：
- 任何 Agent 都不能修改自己负责范围之外的文件。
- 任何 Agent 都不能回退其他人的改动。
- 删除文件、批量重构、改权限、安装依赖、提交代码、联网、处理密钥前必须先请求确认。
- 如果发现边界不够用，停下来报告，不要自行扩大范围。

分工：
- Agent A：只读分析 src/auth 与 tests/auth。
- Agent B：只修改 src/auth/login.ts。
- Agent C：只修改 tests/auth/login.test.ts。
- Agent D：只读审核最终 diff。

请先让 Agent A 输出分析报告，不要进入实现。
```

Claude Code 对话输入：

```markdown
根据 Agent A 的分析，现在进入实现阶段。

Agent B：
- 只修改 src/auth/login.ts。
- 不要格式化其他文件。
- 不要改公共错误码定义，除非先说明必要性。

Agent C：
- 只修改 tests/auth/login.test.ts。
- 补充失败登录的错误码断言。

两个 Agent 完成后请分别列出修改文件和验证命令，不要提交代码。
```

普通终端命令：

```bash
git status --short
git diff --name-only
```

Claude Code 对话输入：

```markdown
请作为审核 Agent，只读检查当前 diff。
如果发现声明范围外的文件被修改，请把文件名列为最高优先级问题。
不要修改文件。
```

## 练习任务

**练习 1：只读探索**

Claude Code 对话输入：

```markdown
请扮演只读探索 Agent，分析当前项目的测试结构。
要求：
- 不修改文件。
- 不安装依赖。
- 不联网。
- 输出测试命令、主要测试目录、潜在风险。
```

**练习 2：文件所有权拆分**

Claude Code 对话输入：

```markdown
把"修复用户资料页表单校验"拆成 3 个子 Agent 任务。
每个任务必须包含：
- 负责文件。
- 禁止修改文件。
- 是否允许运行测试。
- 高风险动作提醒。
不要实际修改文件，只输出任务分配方案。
```

**练习 3：审核 Agent**

普通终端命令：

```bash
git status --short
git diff --stat
```

Claude Code 对话输入：

```markdown
请只读审核当前改动是否符合任务边界。
重点检查：
- 是否改了未授权文件。
- 是否出现删除、权限变更、依赖安装、提交或回退。
- 是否有密钥、token、证书内容进入 diff。
```

**练习 4：冲突处理**

Claude Code 对话输入：

```markdown
假设另一个 Agent 已经修改了 src/api/client.ts，而我也需要改同一个文件。
请给出安全处理流程。
要求：
- 不直接覆盖对方改动。
- 先读取 diff。
- 说明何时需要人工确认。
```

## 验收标准

学完本章，你应该能做到：

- 说清楚什么时候该用子 Agent，什么时候不该用。
- 给每个 Agent 写出明确的角色、输入、输出和禁止事项。
- 用文件清单来表达写入边界。
- 多 Agent 并行时避免覆盖他人的改动。
- 识别高风险动作并设置确认点。
- 用只读审核 Agent 检查 diff、测试缺口和越界修改。
- 在最终总结中列出每个 Agent 修改过的路径和未完成的风险。

## 官方依据

访问日期：2026-04-26。

- Create custom subagents：<https://code.claude.com/docs/en/sub-agents>
- Orchestrate teams of Claude Code sessions：<https://code.claude.com/docs/en/agent-teams>
- Configure permissions：<https://code.claude.com/docs/en/permissions>
- Security：<https://code.claude.com/docs/en/security>

**要点摘录：**

- 子 Agent 在独立上下文中工作，把结果摘要返回主对话，保护主对话上下文不被污染。
- 可配置工具权限、模型（如 Haiku 省成本）、hooks、持久记忆、fork 模式和 worktree 隔离。
- 内置子 Agent：Explore（Haiku 模型，只读探索）、Plan（继承主模型，只读规划）、general-purpose（继承主模型，全工具）。
- Agent teams 适合需要多个 Claude Code 会话直接协作的复杂任务，但目前是实验功能，默认关闭，需设置 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 启用，应谨慎使用。
