---
title: 用 Claude Code + MCP 做自动 Code Review
author: @peterfei
date: 2026-05
stack: Claude Code, GitHub MCP Server, Model Context Protocol
difficulty: 入门
---

## 场景

团队使用 GitHub 管理代码，PR 数量随团队扩张快速增长。纯人工 Code Review 有两个问题：

1. **覆盖不全** — 低级错误（拼写错误、类型不对、缺少边界检查）经常遗漏到线上
2. **Reviewer 疲劳** — 大量重复性的规范检查消耗了 reviewer 的精力，真正需要关注的逻辑变更反而被淹没

目标是：**让 Agent 处理规范性问题，让人 reviewer 只关注逻辑和架构**。

---

## 实现

### 架构

```
GitHub PR → Webhook → Claude Code CLI → MCP(GitHub API) → 读取 diff
                                                        → 按规则审查
                                                        → 评论到 PR
```

### 核心配置

**第一步：安装 Claude Code**

```bash
# 安装 Claude Code（需要 Anthropic 账号）
npm install -g @anthropic-ai/claude-code
```

**第二步：配置 MCP Server**

在 `~/.claude/settings.json` 中配置 GitHub MCP Server：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-token>"
      }
    }
  }
}
```

> GitHub Token 需要 `repo` 权限用于读取 PR 和创建评论。

**第三步：审查脚本**

创建一个本地脚本 `cr.sh`，传入 PR 链接后自动执行审查：

```bash
#!/bin/bash
PR_URL=$1

claude code --print \
  --prompt "你是一个严格的 Code Reviewer。
请审查这个 PR：$PR_URL

审查要求：
1. 只检查规范性问题和潜在 bug，不检查代码风格（那是 linter 的工作）
2. 每一条评论都要给出：问题所在行号 + 问题描述 + 修复建议
3. 如果发现问题，用 GitHub MCP 的 createReviewComment 提交评论
4. 如果一切正常，评论一个 ✅

专注检查：
- 空指针/空值风险
- 并发安全问题
- 资源未释放（文件句柄、数据库连接）
- 边界条件遗漏（空数组、0 值、极端输入）
- 错误处理缺失"
```

**第四步：触发方式**

```bash
# 手动触发（适用于自用）
./cr.sh https://github.com/owner/repo/pull/123
```

也可以集成到 GitHub Actions 自动触发（后续案例详解）。

---

## 效果

在一个 8 人团队的 Go 微服务项目上运行了两周：

| 指标 | 变化 |
|------|------|
| 低级错误漏过率 | 从 ~15% 降到 ~2% |
| 单次 CR 平均耗时 | 45min → 20min |
| Agent 发现的 bug | 12 个（含 2 个并发问题） |
| 误报率 | ~20%（主要在类型推断不准确时） |

**团队反馈**：

> "以前 CR 要专门留出大块时间，现在先让 AI 过一遍，我只需要看它标记的和自己关注的逻辑部分。"
> —— 团队 senior dev

> "有一次 Agent 发现了一个我 review 两轮都没注意到的 nil pointer，之后我就服了。"
> —— 团队 tech lead

---

## 要点

1. **审查范围要克制** — 只做规范性和潜在 bug，不做代码风格。风格问题交给 ESLint/golangci-lint，Agent 做 linter 做不了的事。

2. **误报处理** — 初期误报率较高，通过在 prompt 中加 "不确定就不要报" 可以降低到可接受范围。误报比漏报更伤信任。

3. **人机分工** — Agent 检查"代码是否有问题"，人 reviewer 关注"这个方案是否合理"。两者互补不冲突。

4. **Token 消耗** — 一个大 PR（500+ 行 diff）大约消耗 20K-50K tokens，建议只对关键文件或改动量适中的 PR 启用。

---

## 扩展思路

- 结合 CI：在 GitHub Actions 中自动触发，PR label 标记 `ai-reviewed`
- 按语言定制：给 prompt 加语言特定的审查规则（Go 的 error handling、Python 的 async 模式）
- 增量审查：只审查 diff 中新增的行，忽略删除和格式化变更

---

## 相关资源

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Model Context Protocol 规范](https://modelcontextprotocol.io)
- [GitHub MCP Server](https://github.com/modelcontextprotocol/servers/tree/main/src/github)
- [Claude Code + MCP 配置指南](https://docs.anthropic.com/en/docs/claude-code/mcp)
