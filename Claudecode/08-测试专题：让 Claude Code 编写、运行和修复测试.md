# 测试专题：让 Claude Code 编写、运行和修复测试

这一章讲怎么让 Claude Code 先识别项目的测试框架，再按现有风格补回归测试、跑测试、修失败用例。

## 使用场景

补测试覆盖、给 bug 加回归测试、给新功能写验收测试、理解已有测试结构，或者让 Claude Code 帮你把失败的测试修到合理通过——都适合用本章的方法。重点是确认测试表达的行为是正确的，而不是光让绿灯亮起来。

⚠️ 跑测试可能会写临时文件、访问数据库、启动服务、联网，或者依赖环境变量。别让 Claude Code 擅自装依赖、删测试数据、改权限或提交代码，也别往测试里写真实凭据。

普通终端命令：

```bash
cd /path/to/project
claude
```

Claude Code 对话输入：

```markdown
请先识别这个项目的测试框架和测试约定，不要写代码。
请查看 package.json、测试目录、CI 配置和已有测试样例。
输出：测试框架、命名约定、常用命令、测试数据策略、我应该补测试的位置。
```

## 推荐提示词模板

Claude Code 对话输入：

```markdown
请帮我为这个行为补测试。

目标行为：
[写清楚要验证的行为]

流程：
1. 先只读识别测试框架、断言风格和测试目录
2. 找 2 到 3 个最相似的已有测试作为风格参考
3. 给出测试计划，不要先改
4. 等我确认后，新增或修改最少测试文件
5. 运行最小相关测试
6. 如果失败，先解释失败原因，再修复

约束：
- 不要改生产代码，除非测试暴露真实 bug 且我确认
- 不要安装依赖、联网、删除文件、改权限、提交代码或处理密钥
- 测试中不要使用真实 token、真实账号或生产地址
```

确认后：

Claude Code 对话输入：

```markdown
请按测试计划实现。优先复用现有测试工具和 fixture，不要引入新测试库。完成后运行最小测试命令。
```

## 错误示范

Claude Code 对话输入：

```markdown
帮我把覆盖率提到 90%。
```

**问题在哪：** 目标太大，容易催生低质量的凑覆盖率测试，或者导致大范围修改。

Claude Code 对话输入：

```markdown
测试失败了，你随便改到通过就行。
```

**问题在哪：** "随便改通过"很可能是修改测试期望值来迎合错误行为，掩盖了真正的问题。

普通终端命令：

```bash
npm install some-new-test-library && npm test
```

**问题在哪：** 装新依赖是高风险操作，先确认有没有必要。多数项目应该复用已有的测试框架。

## 正确示范

Claude Code 对话输入：

```markdown
请为"邮箱为空时注册失败"补一个回归测试。
限制范围：优先只改 tests/register.test.ts。
请先查看附近测试风格，输出测试计划，不要立刻编辑。
```

Claude Code 对话输入：

```markdown
计划可以。请只新增这个测试用例，不要重构测试文件。运行注册相关测试即可。
```

普通终端命令：

```bash
npm test -- tests/register.test.ts
```

Python 项目：

```bash
pytest tests/test_register.py
```

## 练习任务

1. 选一个已有函数或接口，找出它目前缺少的边界测试。
2. 让 Claude Code 先识别测试框架和相似测试。
3. 要求它输出测试计划，不要先动手。
4. 确认后补一个最小回归测试。
5. 跑相关测试。
6. 如果测试失败，让 Claude Code 判断是测试写错了、环境有问题，还是生产代码有 bug。

进阶练习：让 Claude Code 列出 5 个边界条件，但只实现其中最重要的 1 个。

## 验收标准

- Claude Code 先识别了测试框架、目录、命名和断言风格。
- 新测试复用了现有工具、fixture 和风格，没有随意引入新依赖。
- 测试验证的是清晰的业务行为，不只是凑行数。
- 跑了最小相关测试，并记录了命令和结果。
- 测试失败时有原因分析，没有直接乱改期望值。
- 没有使用真实密钥、真实账号或生产地址。

## 官方依据

访问日期：2026-04-26。

- Common workflows：<https://code.claude.com/docs/en/common-workflows>
- Permissions：<https://code.claude.com/docs/en/permissions>
- Security：<https://code.claude.com/docs/en/security>
- CLI reference：<https://code.claude.com/docs/en/cli-reference>
