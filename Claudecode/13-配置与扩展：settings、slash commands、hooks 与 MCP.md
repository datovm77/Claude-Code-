# 配置与扩展：settings、skills、hooks 与 MCP

Claude Code 的进阶能力很大一部分来自配置和扩展：`settings` 管全局和项目行为，skills 把常用流程变成可复用命令，hooks 在特定生命周期自动执行检查，MCP 把外部工具和数据源接入 Claude Code。这些机制能提升效率，但也会扩大权限边界，用之前要想清楚。

⚠️ 配置和扩展经常涉及命令执行、联网、外部系统和密钥。任何会删文件、改权限、装依赖、提交代码、联网或处理密钥的配置，都必须先做只读评审，并清楚说明触发条件。

## 使用场景

**settings 适合：**

- 配置默认模型、权限、环境变量、工具策略、项目级规则。
- 固化团队约定，比如禁止某些命令、指定允许的工具。
- 区分用户级（`~/.claude/settings.json`）、项目级（`.claude/settings.json`）和本地个人（`.claude/settings.local.json`）配置。

**skills 适合：**

- 把重复提示词封装成 `/skill-name` 命令，比如 `/review-diff`、`/write-tests`、`/explain-code`。
- 让团队成员用同一套审查和实现流程。

Skills 的标准位置：
- 用户级：`~/.claude/skills/<skill-name>/SKILL.md`
- 项目级：`.claude/skills/<skill-name>/SKILL.md`
- 旧版兼容：`.claude/commands/<name>.md`（仍可使用）

每个 skill 是一个目录，核心文件是 `SKILL.md`，可附带支持文件（模板、示例、脚本）。内置 skills 包括 `/simplify`、`/debug`、`/batch`、`/loop`、`/claude-api`。

**hooks 适合：**

- 在工具调用前后自动检查风险。
- 对写入、命令执行、提交前流程做提醒或阻断。
- 自动运行格式检查、审查脚本或日志记录。

**MCP 适合：**

- 接入外部系统，比如 issue tracker、数据库只读查询、文档库、浏览器或内部工具。
- 给 Claude Code 提供结构化上下文，而不是手动复制粘贴。

**不建议扩展的情况：**

- 还不清楚扩展会获得什么权限。
- MCP 服务需要生产密钥但没有最小权限账号。
- hooks 会自动执行破坏性命令。
- skill 里包含"直接提交""自动删除""自动安装依赖"等动作。

## Skills 详解

SKILL.md 通过 frontmatter 控制行为，核心字段：

| 字段 | 说明 |
|------|------|
| `name` | skill 名称，即 `/name` 命令 |
| `description` | 触发时机说明，让 Claude 知道何时自动调用 |
| `disable-model-invocation: true` | 直接执行，不再调用模型（适合确定性流程） |
| `allowed-tools` | 限定此 skill 可用的工具 |
| `model` | 指定模型，如 `haiku` 降低成本 |
| `effort` | 思考力度：`low / medium / high / xhigh / max` |
| `context: fork` | 在独立 fork 上下文中运行，不污染主对话 |

创建 skill：

普通终端命令：

```bash
mkdir -p .claude/skills/review-diff
```

然后新建 `.claude/skills/review-diff/SKILL.md`：

```markdown
---
name: review-diff
description: 只读审查当前 git diff，重点检查安全风险、测试缺口和提交前问题。当用户说"帮我审查 diff"时调用。
allowed-tools: Read Bash Glob Grep
---

请只读审查当前 git diff。

禁止：修改文件、删除文件、安装依赖、改权限、联网、提交或回退代码、输出密钥

重点检查：密钥/token、删除文件、权限变化、批量重构、测试缺口

输出：
1. 阻塞问题
2. 非阻塞建议
3. 建议测试命令
4. 是否需要人工确认
```

调用方式：

Claude Code 对话输入：

```markdown
/review-diff
```

或直接描述，Claude Code 会根据 description 自动调用匹配的 skill。

## Hooks 详解

Hooks 配置在 settings 的 `hooks` 字段中，格式：

```json
{
  "hooks": {
    "<EventName>": [
      {
        "matcher": "<tool-name-or-*>",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/script.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

**常用 hook 事件（按生命周期顺序）：**

| 事件 | 触发时机 |
|------|---------|
| `SessionStart` | 会话启动时 |
| `UserPromptSubmit` | 用户提交提示词后 |
| `PreToolUse` | 工具调用前 |
| `PermissionRequest` | 权限申请时 |
| `PostToolUse` | 工具调用成功后 |
| `PostToolUseFailure` | 工具调用失败后 |
| `SubagentStart` | 子 Agent 启动时 |
| `SubagentStop` | 子 Agent 结束时 |
| `Stop` | 会话结束时 |
| `FileChanged` | 文件被修改时 |
| `PreCompact` / `PostCompact` | 压缩上下文前后 |
| `SessionEnd` | 会话完全结束时 |

**退出码含义：**
- `0`：允许通过
- `2`：阻断（根据事件不同效果不同，如在 `PreToolUse` 时拒绝工具调用）
- 其他：非阻塞错误

**用 `/hooks` 命令管理：** 在会话内输入 `/hooks` 打开交互式 hooks 管理界面，可以查看、添加、编辑和禁用 hooks。

**阻断 `rm -rf` 示例：**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/block-dangerous.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/block-dangerous.sh`：

```bash
#!/bin/bash
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -qE 'rm -rf|chmod -R|git reset --hard'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "危险命令被 hook 拦截，请人工确认后再执行"
    }
  }'
else
  exit 0
fi
```

## 推荐提示词模板

Claude Code 对话输入：

```markdown
请只读分析当前项目是否适合加入 Claude Code settings。
要求：
- 不修改文件。
- 不安装依赖。
- 不联网。
- 输出：建议配置项、风险、是否适合放在用户级或项目级。
```

Claude Code 对话输入：

```markdown
请帮我设计一个 /review-diff skill。
目标：审查当前 git diff。
要求：
- 默认只读。
- 不修改文件。
- 不提交代码。
- 检查密钥、删除文件、权限变化、依赖安装、批量重构。
- 输出中文 Markdown。
先输出 SKILL.md 草案，不要写入文件。
```

Claude Code 对话输入：

```markdown
请评估是否应该为本项目添加 hooks。
要求：
- 只读分析。
- 列出适合的触发点和对应事件名。
- 每个 hook 都说明风险。
- 不要创建或修改配置文件。
```

Claude Code 对话输入：

```markdown
请为接入一个只读文档 MCP 设计安全方案。
要求：
- 不包含真实密钥。
- 说明最小权限原则。
- 说明联网审核点。
- 说明如何验证 MCP 只提供只读能力。
```

普通终端命令：

```bash
claude --help
git diff -- .claude/
```

用途：查看当前版本支持的 CLI 参数；审查 `.claude/` 配置变更。

## 错误示范

Claude Code 对话输入：

```markdown
帮我加一个 hook，每次运行命令前自动 chmod -R 777 .
```

**问题在哪：** 改权限本就是高风险操作，`chmod -R 777 .` 会破坏安全边界，自动触发会把事故范围放大。

Claude Code 对话输入：

```markdown
创建一个 MCP，连接生产数据库，给 Claude 完整读写权限。
```

**问题在哪：** 生产数据库加写权限，风险极高。缺少只读账号、审计机制和查询限制，可能暴露隐私数据或密钥。

Claude Code 对话输入：

```markdown
做一个 /fix-all skill，自动重构、安装依赖、删除无用文件并提交。
```

**问题在哪：** 把多个高风险操作打包进一个快捷命令，没有人工确认，没有测试环节。

```json
{
  "permissions": {
    "allow": ["Bash(*)"]
  }
}
```

**问题在哪：** 权限放得太宽，危险命令会直接绕过人工确认。应该按命令、目录和任务最小化授权。

## 正确示范

**settings 思路示例：**

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "Bash(git status)",
      "Bash(git diff *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(chmod -R *)",
      "Bash(git reset --hard *)",
      "Bash(git push *)"
    ]
  }
}
```

说明：`allow` 放白名单，`deny` 明确禁止危险命令。配置前先用 `claude doctor` 确认版本，避免使用已废弃字段。

## 练习任务

**练习 1：settings 评估**

Claude Code 对话输入：

```markdown
请只读检查当前项目是否已有 .claude 配置。
输出：
- 发现的配置文件。
- 可能影响权限的字段。
- 是否存在危险命令或密钥。
不要修改文件。
```

**练习 2：skill 设计**

Claude Code 对话输入：

```markdown
请设计一个可通过 /test-plan 调用的 skill，用于根据当前 diff 生成测试计划。
要求：
- 不运行测试。
- 不修改文件。
- 输出手动测试和自动测试建议。
先输出 SKILL.md 草案，不要写入文件。
```

**练习 3：hook 风险评审**

Claude Code 对话输入：

```markdown
请评审一个计划中的 PreToolUse hook：在每次 Bash 工具调用前检查命令是否包含 rm、chmod、git reset、npm install。
请说明：
- 能降低什么风险。
- 可能误报什么。
- 是否应该阻断还是只提醒。
- 对应的退出码应该是几。
```

**练习 4：MCP 最小权限设计**

Claude Code 对话输入：

```markdown
请为一个只读 issue tracker MCP 写接入方案。
要求：
- 不包含真实 token。
- 限制只读。
- 说明如何撤销访问。
- 说明联网审核点。
```

## 验收标准

学完本章，你应该能做到：

- 区分 settings、skills、hooks、MCP 各自适合什么场景。
- 知道 skills 的标准目录结构（`.claude/skills/<name>/SKILL.md`）和旧版兼容路径。
- 写出默认只读的 SKILL.md 草案，包含正确的 frontmatter。
- 识别 hooks 中自动执行危险命令的风险，并说出退出码含义。
- 列出至少 5 个常用 hook 事件名称和触发时机。
- 为 MCP 接入设计最小权限、联网审核和密钥保护方案。
- 修改 `.claude/` 配置前先审查 diff。

## 官方依据

访问日期：2026-04-26。

- Claude Code settings：<https://code.claude.com/docs/en/settings>
- Commands：<https://code.claude.com/docs/en/commands>
- Extend Claude with skills：<https://code.claude.com/docs/en/skills>
- Hooks reference：<https://code.claude.com/docs/en/hooks>
- Automate with hooks（guide）：<https://code.claude.com/docs/en/hooks-guide>
- Connect Claude Code to tools via MCP：<https://code.claude.com/docs/en/mcp>
- Configure permissions：<https://code.claude.com/docs/en/permissions>
- Security：<https://code.claude.com/docs/en/security>
