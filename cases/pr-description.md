---
title: 用 Claude Code 自动生成 PR 描述
author: @peterfei
date: 2026-05
stack: Claude Code, Git, GitHub CLI
difficulty: 入门
---

## 场景

团队在 Code Review 时最常遇到的问题：

1. PR 描述写得太简单（"修了个 bug"），reviewer 不知道改了啥
2. 描述和实际代码不一致（改之前写的，改完忘了更新）
3. 没人愿意写 PR 描述，觉得浪费时间

目标是：**根据 git diff 自动生成 PR 描述，人工确认后提交**。

---

## 实现

### 架构

```
git diff → Claude Code → 分析变更 → 生成结构化 PR 描述 → gh pr create
```

### 核心脚本

创建一个 git 别名或 npm script：

```bash
#!/bin/bash
# pr-gen.sh — 生成 PR 描述并创建 PR

BRANCH=$(git rev-parse --abbrev-ref HEAD)
DIFF=$(git diff main...HEAD --stat)

# 用 Claude Code 分析 diff 并生成描述
claude code --print --prompt "
你是一个技术团队的 PR 描述生成器。

当前分支：$BRANCH

变更文件统计：
$DIFF

请分析 git diff（会提供给你）并生成 PR 描述。

输出格式（Markdown）：

## Summary
一句话概括这次 PR 做了什么

## Changes
- filename: 改了啥（一句话）
- filename: 改了啥

## Why
为什么做这个变更

## Testing
如何测试

## Related
关联的 Issue（如果有）

注意事项：
- 描述要具体，不要写"优化性能"，要写"将列表查询从 O(n²) 优化为 O(n log n)"
- 不要写 diff 中不存在的内容
- 如果检测到潜在的 breaking change，在末尾标注 ⚠️
"
```

### 使用方式

```bash
# 一行命令创建 PR
git checkout -b feat/add-login
# ... 写代码 ...
git add -A && git commit -m "feat: add login"
./pr-gen.sh
# → 自动生成描述 → 打开编辑器确认 → 提交 PR
```

### 配合 GitHub CLI

生成描述后可以直接用 `gh` 创建 PR：

```bash
claude code --print --prompt "..." > /tmp/pr-body.md
gh pr create --title "$(head -1 /tmp/pr-body.md)" --body "$(cat /tmp/pr-body.md)"
```

---

## 效果

在团队试用 3 周（6 人团队，约 40 个 PR）：

| 指标 | 变化 |
|------|------|
| 有描述的 PR 比例 | 35% → 95% |
| 平均 PR 描述质量评分 | 5/10 → 8/10 |
| 创建 PR 耗时 | 5min → 30s |
| CR 首次回复时间 | 4h → 2h（描述清晰后 reviewer 更愿意看） |

**团队反馈**：

> "之前写 PR 描述像交作业，现在像有个助手帮我写草稿我再改改。"
> —— 团队成员

> "描述变好了之后，review 的时候不用再自己跑一遍代码猜意图了。"
> —— 团队 tech lead

---

## 要点

1. **使用 `--print` 模式** — Claude Code 的 `--print` 模式直接输出结果到 stdout，适合脚本化调用。不加 `--print` 会进入交互模式。

2. **分支名提取信息** — 分支名 `feat/add-login` 本身就包含 type 和 scope 信息，可以告诉 Agent 按 Conventional Commits 识别。

3. **人工确认环节** — 自动生成的描述建议在创建 PR 前看一眼。Agent 偶尔会脑补 diff 中没有的变更。

4. **模版一致性** — 团队应该统一 PR 描述模板。把模板写死在 prompt 里，所有人的 PR 描述格式一致，reviewer 不用适应不同风格。

---

## 扩展思路

- 集成 Conventional Commit：从 commit 消息推断 type（feat/fix/refactor）
- 自动检测 breaking change：如果修改了 public API 签名，自动标注
- 生成 CHANGELOG 条目：每次 PR 合并后自动收集描述，生成发布日志

---

## 相关资源

- [Conventional Commits](https://www.conventionalcommits.org)
- [GitHub CLI](https://cli.github.com)
- [Claude Code --print 模式](https://docs.anthropic.com/en/docs/claude-code/overview)
