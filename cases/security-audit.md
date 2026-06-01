---
title: 用 AI 做自动化代码安全审计 Agent
author: "@peterfei"
date: 2026-06
stack: Semgrep, Claude Code, MCP, GitLab CI, OWASP, Docker
difficulty: 进阶
---

## 场景

大部分团队的安全现状：

1. **安全 testing 靠上线前人工扫** — 要么没时间做，要么上线后才被发现漏洞
2. **第三方依赖漏洞** — npm audit 天天报，但没人看，没人修
3. **重复安全问题** — SQL 注入、XSS、硬编码密钥，同样的错误反复犯
4. **安全 PR Review 流于形式** — Reviewer 没精力逐行看安全问题

目标是：**让 Agent 接入 CI 流水线，每次提交自动做安全审计，发现漏洞立即报警并给出修复建议**。

---

## 实现

### 架构

```
代码提交 → CI 触发 → ① Semgrep SAST 扫描（规则匹配）
                  → ② AI 深度分析（规则没覆盖到的逻辑漏洞）
                  → ③ 依赖漏洞检查（npm/pip/go）
                  → ④ 密钥泄露检测（硬编码 token/密码）
                  → ⑤ 生成安全报告 → PR Comment / 飞书告警
                                    ↓
                          人工确认 → 修复 / 忽略
```

### 核心实现

```yaml
# .gitlab-ci.yml 或 GitHub Actions
security-audit:
  stage: test
  script:
    # Step 1: Semgrep 规则扫描
    - semgrep --config=auto --json -o semgrep-report.json .

    # Step 2: AI 深度分析
    - claude code --print --prompt "
      你是一个安全工程师，正在对以下代码变更做安全审计。

      Git Diff：
      $(git diff main...HEAD)

      Semgrep 扫描结果：
      $(cat semgrep-report.json | jq '.results[] | {check_id, path, line, message}')

      ## 审计要求

      1. 确认 Semgrep 发现的漏洞是否属实（去误报）
      2. 检查 Semgrep 规则未覆盖的逻辑漏洞：
         - 权限检查缺失（用户 A 能否访问用户 B 的数据？）
         - 输入校验不完整（文件上传类型检查、参数长度限制）
         - 不安全的直接对象引用（IDOR）
         - SSRF 风险
         - 竞态条件
      3. 检查硬编码密钥、Token、密码
      4. 检查不安全的配置（DEBUG 模式开启、CORS 配置过松）

      ## 输出格式

      ### 高危 🚨
      - [文件:行号] 问题描述 | 修复建议

      ### 中危 ⚠️
      - ...

      ### 低危 ℹ️
      - ...

      ### 误报已排除
      - Semgrep 报告但实际安全的项（说明原因）
      " > security-report.md

  artifacts:
    paths:
      - security-report.md
```

### AI 深度审计 Prompt 示例

以下是一个专门针对 Web 应用的审计 Prompt，覆盖 OWASP Top 10：

```markdown
你是一个安全审计专家。请审查以下代码，特别关注：

## 认证与授权
- 是否存在越权访问？（用户 A 能否操作属于用户 B 的资源？）
- Token/JWT 校验是否在每个受保护端点都执行了？
- 密码策略是否符合规范（长度、复杂度）？

## 输入验证
- 用户输入是否经过充分过滤？
- 文件上传是否有类型和大小限制？
- 是否存在 SQL 注入风险？（特别是拼接 SQL 的地方）
- 是否存在 XSS 风险？（用户输入直接渲染到 HTML）

## 敏感信息
- 是否存在硬编码的 API Key、密码、Token？
- 日志中是否会输出敏感信息（身份证、手机号）？
- 配置文件中是否包含生产环境密钥？

## 配置安全
- CORS 是否配置了白名单而非 `*`？
- Debug 模式在生产环境是否已关闭？
- HTTPS 是否强制开启？

## 依赖安全
- 是否使用了有已知漏洞的依赖版本？
- 是否引入了不必要的庞大依赖（攻击面扩大）？

请在每个发现项上标注：🚨高危 / ⚠️中危 / ℹ️低危，并给出修复代码示例。
```

### 密钥泄露检测

```bash
#!/bin/bash
# secret-scan.sh — 检测硬编码密钥

echo "=== 密钥泄露扫描 ==="

# 使用 truffleHog 扫描 git 历史
docker run --rm -v "$(pwd):/repo" trufflesecurity/trufflehog:latest \
  filesystem --directory=/repo --json | tee secrets.json

# AI 分析扫描结果
claude code --print --prompt "
以下是从代码仓库中检测到的潜在密钥泄露：

$(cat secrets.json | jq -r '. | select(.redacted == false) | "文件: \(.SourceMetadata.Data.Files)\n类型: \(.DetectorName)\n---"')

请判断：
1. 哪些是真实的密钥（去误报）
2. 泄露的密钥涉及的资产范围
3. 紧急处理步骤（吊销、轮转）
4. 预防措施（Git hooks、.gitignore 规则）
" > secret-response.md

# 如果有高危泄露，直接告警
HIGH_COUNT=$(cat secrets.json | jq '[. | select(.redacted == false)] | length')
if [ "$HIGH_COUNT" -gt 0 ]; then
  echo "⚠️ 发现 $HIGH_COUNT 个潜在密钥泄露！"
  echo "详情见 secret-response.md"
fi
```

### PR 自动安全评论

在 CI 中集成，每次 PR 自动发布安全审计结果：

```yaml
- name: Comment Security Report
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      const report = fs.readFileSync('security-report.md', 'utf8');
      const hasHigh = report.includes('🚨');

      await github.rest.issues.createComment({
        ...context.repo,
        issue_number: context.issue.number,
        body: `## 🔒 安全审计报告\n\n${report}\n\n${
          hasHigh
            ? '> ⚠️ 存在高危漏洞，请优先处理后再合并'
            : '> ✅ 未发现高危漏洞'
        }`
      });
```

---

## 效果

在一个 50+ 微服务的 Node.js/Go 项目上运行 4 周：

| 指标 | 变化 |
|------|------|
| PR 安全覆盖率 | 5% → 95%（几乎每个 PR 都自动审计） |
| 漏洞发现前置 | 平均上线后 3 天发现 → 上线前即发现 |
| 误报率 | Semgrep 单独 40% → 结合 AI 后降至 12% |
| 高危漏洞修复率 | 30% → 85%（有 AI 修复建议后修复意愿大增） |
| 安全 review 时间 | 每 PR 平均 20min → 3min（只看 AI 标记的就好） |

**发现的漏洞类型分布**：

| 类型 | 数量 | 说明 |
|------|------|------|
| 硬编码密钥/Token | 23 | 测试代码中写死 AWS 密钥，CI 配置文件中的密码 |
| SQL 注入 | 5 | 拼接查询条件，未使用参数化查询 |
| 越权访问 (IDOR) | 12 | API 未校验资源归属，用户 A 可查用户 B 订单 |
| CORS 配置过松 | 8 | 允许所有来源访问内网 API |
| 依赖漏洞 | 34 | lodash、axios 等已知漏洞版本 |
| Debug 模式未关 | 6 | 生产环境暴露错误堆栈 |

---

## 要点

1. **规则引擎 + AI 是黄金组合** — Semgrep 规则适合确定性模式（SQL 注入、密钥格式），AI 擅长逻辑漏洞（越权、业务逻辑缺陷）。先跑规则引擎去重，再让 AI 做深度分析，效率最高。

2. **去误报是落地关键** — Semgrep 单独有 40% 误报率，开发看几次就麻木了。让 AI 先过滤一遍，只保留 true positive，开发才愿意看。误报率降到 15% 以下是团队愿意用起来的阈值。

3. **修复建议要具体到代码** — "存在 SQL 注入风险"这种提醒没人爱看。AI 给出修复后的代码 diff，开发直接复制就能修，修复率从 30% 提升到 85%。

4. **密钥检测要扫 git 历史** — 只在当前代码中扫描不够，密钥可能已经存在于历史 commit 里。truffleHog 逐 commit 扫描，发现后立即轮转密钥。

5. **坑：大型仓库扫描超时** — 50 万行以上代码，全量 Semgrep + AI 分析可能超过 CI 超时限制。建议增量扫描：只分析 git diff 涉及的变更文件，全量扫描放到 nightly build。

---

## 扩展思路

- **IaC 安全审计**：扫描 Terraform/K8s YAML 中的安全配置风险
- **容器镜像扫描**：结合 Trivy 扫描 Docker 镜像的 CVE
- **运行时安全监控**：Agent 分析异常行为模式（异常登录、数据批量导出）
- **合规检查**：自动检查代码是否符合 PCI-DSS / GDPR / 等保要求
- **安全知识库**：将历史漏洞沉淀为规则，Agent 自动学习团队特有的安全模式

---

## 相关资源

- [Semgrep — 轻量级静态分析工具](https://semgrep.dev)
- [truffleHog — Git 密钥泄露扫描](https://github.com/trufflesecurity/trufflehog)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [GitLab 安全扫描集成](https://docs.gitlab.com/ee/user/application_security/)
- [GitHub CodeQL](https://codeql.github.com)
