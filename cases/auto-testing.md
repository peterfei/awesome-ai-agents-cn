---
title: 用 Claude Code + Playwright 做自动化测试 Agent
author: "@peterfei"
date: 2026-06
stack: Claude Code, Playwright, MCP, GitHub Actions, Node.js
difficulty: 进阶
---

## 场景

大多数团队在测试上投入不足，原因很现实：

1. **写测试太慢** — 一个前端页面从写完到补完 E2E 测试，至少多花 1-2 小时
2. **测试维护成本高** — UI 一改，测试就挂，没人愿意修
3. **覆盖率形同虚设** — 核心流程 80%，边界情况几乎为 0
4. **测试数据难准备** — 构造各种状态的数据比写测试本身还麻烦

目标是：**让 Agent 根据功能描述自动生成并执行测试，开发者只需要审查结果**。

---

## 实现

### 架构

```
功能描述 → Claude Code → ① 分析功能，确定测试场景
                        → ② 生成 Playwright 测试代码
                        → ③ 执行测试，收集结果
                        → ④ 分析失败原因，自动修复
                        → ⑤ 输出测试报告
                                    ↓
                         人工审查 ←—— 生成的测试 + 报告
```

### 核心配置

**第一步：安装 Playwright MCP**

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-playwright"]
    }
  }
}
```

**第二步：测试生成 Prompt**

创建一个 `test-agent.md`：

```markdown
你是一个 QA 工程师，负责为以下功能编写 E2E 测试。

## 功能描述
{feature_description}

## 测试要求
1. 覆盖正常流程、异常流程、边界情况
2. 每个测试用例一个独立的 `test` 块
3. 使用 Page Object Model 组织代码
4. 测试数据使用 fake 数据生成
5. 关键步骤添加 `await page.screenshot()` 便于排查失败

## 输出格式
- 测试文件路径
- 测试用例清单（含覆盖的场景说明）
- 需要手动确认的步骤
```

### 核心脚本

```bash
#!/bin/bash
# test-gen.sh — AI 测试生成与执行

FEATURE="$1"
FEATURE_DIR="$2"

# Step 1: 生成测试
claude code --print --prompt "
$(cat test-agent.md)

请为以下功能生成 Playwright 测试：
功能：$FEATURE
目录：$FEATURE_DIR
" > "$FEATURE_DIR/__tests__/generated.spec.ts"

# Step 2: 执行测试
npx playwright test "$FEATURE_DIR/__tests__/generated.spec.ts" \
  --reporter=json 2>/dev/null | tee /tmp/test-result.json

# Step 3: 分析失败
FAILURES=$(cat /tmp/test-result.json | jq '.suites[0].specs[] | select(.ok == false)')

if [ -n "$FAILURES" ]; then
  echo "检测到测试失败，正在分析..."

  claude code --print --prompt "
  以下测试用例执行失败，请分析失败原因并修复测试代码。

  失败详情：
  $(echo "$FAILURES" | jq -r '.title + ": " + .tests[0].errors[0].message')

  测试文件内容：
  $(cat "$FEATURE_DIR/__tests__/generated.spec.ts")

  请输出修复后的完整测试文件。
  " > "$FEATURE_DIR/__tests__/generated.spec.ts"

  # 重新执行
  npx playwright test "$FEATURE_DIR/__tests__/generated.spec.ts"
fi

# Step 4: 生成报告
echo "=== 测试报告 ==="
echo "通过: $(cat /tmp/test-result.json | jq '[.suites[0].specs[] | select(.ok)] | length')"
echo "失败: $(cat /tmp/test-result.json | jq '[.suites[0].specs[] | select(.ok == false)] | length')"
```

### CI 集成

在 GitHub Actions 中配置自动测试生成：

```yaml
name: AI Test Generation
on:
  pull_request:
    paths:
      - 'src/**/*.tsx'
      - 'src/**/*.ts'

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npx playwright install --with-deps

      - name: AI Generate Tests
        run: |
          claude code --print --prompt "
            分析以下 PR 变更，生成对应的 Playwright 测试
            $(git diff main...HEAD --stat)
          " > e2e/generated/pr-${PR_NUMBER}.spec.ts
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Run Generated Tests
        run: npx playwright test e2e/generated/

      - name: Comment PR with Results
        uses: actions/github-script@v7
        with:
          script: |
            // 将测试结果以 comment 形式发布到 PR
```

---

## 效果

在一个中大型 React 项目（200+ 页面，500+ API 接口）上试用 3 周：

| 指标 | 变化 |
|------|------|
| E2E 测试覆盖率 | 23% → 68% |
| 测试编写时间 | 平均 45min/功能 → 5min/功能 |
| 测试维护时间 | 每周 8h → 每周 2h |
| CI 中漏测的 bug | 减少 40% |
| 开发者对测试的满意度 | 3/10 → 7/10 |

**典型场景效果**：

| 场景 | 人工 | Agent | 说明 |
|------|------|-------|------|
| 表单验证测试 | 30min | 3min | 覆盖空值、格式错误、边界值 |
| 分页列表测试 | 20min | 2min | 空数据、单页、多页、搜索过滤 |
| 权限测试 | 45min | 5min | 不同角色看到不同菜单/按钮 |
| 支付流程测试 | 60min | 8min | 成功、失败、超时、退款各状态 |

**翻车案例**：
- Agent 生成的测试有时过于乐观（假设 API 永远返回正确数据），漏掉了网络错误处理
- 复杂拖拽交互（DnD）生成的定位器不稳定，需要人工调整
- 生成的测试数据有时不符合业务规则（如生成 200 岁的用户）

---

## 要点

1. **Page Object 是关键** — 不生成 POM 的情况下，Agent 生成的测试选择器散落在各个 test 中，UI 一改全挂。先让 Agent 生成 Page Object，再基于 POM 生成测试。

2. **截图对比定位失败** — 测试失败时，让 Agent 看截图比看报错信息更有用。在每个关键操作后加 `page.screenshot()`，失败时把截图发给 Agent 分析。

3. **避免重复生成** — 第一次生成后，后续 PR 只生成增量测试（只测试变更部分）。用 `git diff` 告诉 Agent 改了哪些文件，避免每次重新生成全部测试。

4. **自愈机制** — Playwright 自身有 `auto-waiting` 机制，但 Agent 生成的测试还可以加一层：如果 `locator` 找不到，尝试用 `getByRole`/`getByText` 等替代策略重试。

5. **坑：动态 ID** — Ant Design / Element Plus 等组件库的动态 ID（`ant-btn-123`）每次渲染不同。必须在 prompt 里明确要求只用 `getByRole`、`getByText`、`getByTestId`，禁止使用动态 ID 或 CSS 路径。

---

## 扩展思路

- **视觉回归测试**：结合 Percy/Chromatic 做截图对比
- **API 测试生成**：从 OpenAPI 规范自动生成接口测试
- **性能测试生成**：在生成的 E2E 测试中嵌入 Lighthouse 性能检测
- **测试数据工厂**：Agent 自动生成符合业务规则的测试数据工厂函数
- **测试优先级排序**：根据代码变更影响范围，自动排序测试执行顺序

---

## 相关资源

- [Playwright 官方文档](https://playwright.dev)
- [@anthropic/mcp-playwright](https://github.com/anthropics/mcp-playwright)
- [Page Object Model 模式](https://playwright.dev/docs/pom)
- [Ant Design 可访问性定位](https://ant.design/docs/react/accessibility-cn)
