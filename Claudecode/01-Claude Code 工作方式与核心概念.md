# Claude Code 工作方式与核心概念

Claude Code 是跑在你开发环境里的编程助理。你用自然语言描述任务，它负责读项目、想步骤、改文件、跑命令，遇到需要权限的地方会先问你。要把它用好，关键是搞清三件事：它看到了什么、它能调用什么工具、哪些动作需要你点头。

本章只讲环境自检和常见问题排查，不展开安装教程。

⚠️ 涉及删文件、大范围重构、改权限、装依赖、提交代码、联网、碰密钥时，要求 Claude Code 先说明计划和影响范围，等你确认后再动手。

## 使用场景

下面这些事情很适合用 Claude Code 来做：

- 让它读项目、解释结构。
- 让它改几个文件，然后给你看 diff。
- 让它跑测试、格式化、lint 等检查命令。
- 用 `/help` 看可用命令，用 `claude doctor` 检查环境，用 `/permissions` 管权限。

**核心心智模型：**

- 对话框里输入的是任务说明，不是 shell 命令。
- `pwd`、`git status`、`claude --version` 这类是普通终端命令，由 shell 执行。
- Claude Code 会调用工具来读文件、改文件、跑命令。
- 权限决定了它能不能执行某些命令或访问某些资源。
- 最终拍板的是你——确认范围、审 diff、决定提不提交。

## 环境自检

普通终端命令：

```bash
claude --version
claude doctor
pwd
```

Windows PowerShell 普通终端命令：

```powershell
claude --version
claude doctor
Get-Location
```

进项目目录后再启动：

普通终端命令：

```bash
cd /path/to/your-project
pwd
claude
```

启动会话后可以输入：

Claude Code 对话输入：

```markdown
/help
```

Claude Code 对话输入：

```markdown
/permissions
```

Claude Code 对话输入：

```markdown
/config
```

说明：`claude doctor` 是普通终端命令，在启动 Claude Code 前在终端里运行；`/help`、`/permissions`、`/config` 是会话内命令，在对话框输入。

**常见不可用原因：**

- 终端找不到 `claude` 命令——多半是 PATH 没配好，或者终端没重启。
- 没登录账号，或者当前账号没有可用订阅、Console 额度、企业权限或第三方配置。
- 当前目录不是项目根目录，Claude Code 拿到的上下文不完整。
- Git、Node、Python、包管理器或项目依赖缺失，导致检查命令跑不了。
- 权限设置挡住了文件写入、命令执行、联网或路径访问。
- 公司代理、防火墙、证书或网络策略导致连不上服务。

## CLAUDE.md 与 auto memory

Claude Code 有两种方式记住项目信息，跨会话持续生效：

**CLAUDE.md：** 你主动维护的项目说明文件，放在 `CLAUDE.md`（项目根目录，建议提交到 git 团队共享）、`.claude/CLAUDE.md` 或 `CLAUDE.local.md`（个人本地，加入 `.gitignore`）。每次会话启动，Claude Code 会自动加载它。

典型内容：测试命令、代码风格约定、重要目录说明、禁止事项。

例子：

```markdown
# 项目说明
- 测试命令：npm test -- --runInBand
- 不要修改 src/legacy 目录
- API handlers 在 src/api/handlers/
```

**auto memory：** Claude Code 在会话过程中自动记录你的纠正和偏好，存储到 `~/.claude/projects/<project>/memory/`，下次会话自动应用。用 `/memory` 命令查看和编辑。

如果 Claude Code 反复犯同一个错误，或者你发现自己每次都要输入同样的约定，就应该把它写进 `CLAUDE.md`。

Claude Code 对话输入：

```markdown
/init
```

用途：在当前目录生成初始 `CLAUDE.md`，包含项目基础信息。

## 推荐提示词模板

Claude Code 对话输入：

```markdown
请先不要修改文件。请检查当前 Claude Code 会话的工作目录和项目上下文。

请输出：
1. 当前目录是否像项目根目录。
2. 你能看到的关键文件。
3. 你建议我先运行哪些普通终端命令验证环境。
4. 有哪些权限或安全风险需要确认。
```

Claude Code 对话输入：

```markdown
请解释你准备如何完成这个任务。执行任何文件修改、安装依赖、联网、删除文件、改权限、提交代码前，都必须先停下来让我确认。

任务：阅读当前项目并找出测试命令。
限制：只读分析，不修改文件。
```

Claude Code 对话输入：

```markdown
请帮我判断当前报错是来自 Claude Code 环境、项目依赖、权限设置还是网络连接。请先列排查顺序，再告诉我每一步要运行的普通终端命令。
```

## 错误示范

Claude Code 对话输入：

```markdown
你自己看着办，能跑起来就行，缺什么就装什么。
```

**问题在哪：** "缺什么就装什么"等于给了装依赖的权限，但你根本不知道它会装什么。

Claude Code 对话输入：

```markdown
所有权限都给你，直接修。
```

**问题在哪：** 权限放得太宽，出了事很难追溯。没限定能不能联网、删文件、改权限。

## 正确示范

Claude Code 对话输入：

```markdown
请只读检查当前项目是否具备运行条件，不要修改文件。

请检查：
- 当前目录
- 项目语言和依赖文件
- 可能的测试命令
- Claude Code 权限是否足够完成只读分析

输出：
- 发现的问题
- 建议我手动执行的普通终端命令
- 哪些动作需要额外确认
```

普通终端命令：

```bash
pwd
git status
ls
claude --version
claude doctor
```

Claude Code 对话输入：

```markdown
请基于环境检查结果解释问题来源。除非我明确同意，不要修改配置文件、不要安装依赖、不要联网下载任何内容。
```

## 权限模式速查

在会话内按 Shift+Tab 可以切换权限模式。常用模式：

- `default`：每次高风险操作都需要确认。
- `acceptEdits`：自动接受文件编辑，命令执行仍需确认。适合你已信任修改方向的时候。
- `plan`：只读探索和规划，不编辑文件。
- `bypassPermissions`：跳过所有确认，仅用于受控的 CI/CD 环境。

也可以在 CLI 启动时指定：

普通终端命令：

```bash
claude --permission-mode plan
```

## 练习任务

1. 在一个练习项目根目录运行 `claude doctor`，检查环境状态。
2. 启动 Claude Code 后输入 `/help`，看看有哪些可用命令。
3. 输入 `/permissions`，查看权限设置。
4. 输入 `/init`，生成一个初始 `CLAUDE.md`，并阅读它。
5. 让 Claude Code 只读分析项目结构，并指出它没把握的信息。

普通终端命令：

```bash
claude --version
claude doctor
pwd
git status
```

Claude Code 对话输入：

```markdown
请用"确定 / 不确定 / 需要我确认"三类整理你对当前项目的判断。不要修改文件。
```

## 验收标准

学完本章，你应该能做到：

- 分清普通终端命令（如 `claude doctor`）和 Claude Code 会话内命令（如 `/help`）。
- 会用 `claude --version`、`claude doctor` 做基础自检，会话内用 `/help`、`/permissions`、`/config` 获取帮助和管理权限。
- 确认 Claude Code 是否在正确的项目目录里运行。
- 理解 `CLAUDE.md` 的作用并知道什么时候该更新它。
- 会用 Shift+Tab 切换权限模式。
- 列举 Claude Code 不可用的常见原因。
- 在高风险操作前要求 Claude Code 先暂停。

## 官方依据

访问日期：2026-04-26。

- Claude Code Overview：<https://code.claude.com/docs/en/overview>
- Quickstart：<https://code.claude.com/docs/en/quickstart>
- CLI reference：<https://code.claude.com/docs/en/cli-reference>
- Commands：<https://code.claude.com/docs/en/commands>
- Store instructions and memories (CLAUDE.md)：<https://code.claude.com/docs/en/memory>
- Permission modes：<https://code.claude.com/docs/en/permission-modes>
- Configure permissions：<https://code.claude.com/docs/en/permissions>
- Security：<https://code.claude.com/docs/en/security>
