---
title: 用 Claude Code + MCP 做日志分析故障排查 Agent
author: "@peterfei"
date: 2026-06
stack: Claude Code, MCP, Shell, Kubernetes
difficulty: 进阶
---

## 场景

线上服务出问题时的典型流程：

1. 收到告警（PagerDuty/飞书/钉钉）
2. SSH 到服务器，翻日志
3. grep / awk / sed 查关键字
4. 找到线索，去 Grafana 看监控
5. 定位到问题，修复

这个过程每一步都不难，但每一次都要重复。尤其是在凌晨被告警叫醒时，大脑还没完全开机，翻日志的效率更低。

目标是：**让 Agent 能在终端里直接分析日志，执行排查脚本，给出故障原因推测。**

---

## 实现

### 架构

```
告警触发 → Claude Code (Shell MCP) → ① 读取最近的日志
                                    → ② 分析错误模式
                                    → ③ 执行诊断脚本
                                    → ④ 输出故障报告
```

### 核心配置

**第一步：配置 Shell MCP**

在 `~/.claude/settings.json` 中启用 Shell MCP：

```json
{
  "mcpServers": {
    "shell": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-shell"],
      "env": {
        "ALLOWED_COMMANDS": "tail,grep,awk,sed,curl,kubectl,docker,cat,head,wc,sort,uniq,jq"
      }
    }
  }
}
```

> 安全限制：只允许白名单内的命令，不允许 `rm`、`sudo`、`ssh` 等危险操作。

**第二步：诊断提示词**

创建一个诊断脚本 `diagnose.sh`：

```bash
#!/bin/bash
# 用法: claude code --prompt "$(cat diagnose-prompt.md)"

SERVICE=$1
TIME_RANGE=$2

cat << PROMPT
你是一个 SRE 运维工程师，正在排查 $SERVICE 服务的问题。

可用的命令：
- tail -n 200 /var/log/$SERVICE/error.log  — 查看最近错误日志
- tail -n 500 /var/log/$SERVICE/app.log    — 查看最近应用日志
- kubectl get pods -n $SERVICE             — 查看 pod 状态
- kubectl logs --tail=100 -n $SERVICE      — 查看 pod 日志
- curl -s http://localhost:9090/metrics    — 查看 metrics
- curl -s http://localhost:8080/health     — 健康检查

诊断流程：
1. 先看健康检查端点，确认服务是否在运行
2. 查看错误日志，定位异常堆栈
3. 查看最近时间段的访问日志，寻找异常模式
4. 如果是性能问题，查看 metrics
5. 如果是崩溃问题，查看 pod 状态和重启次数

输出格式：
## 故障概要
[一句话描述问题]

## 根因分析
[定位到的具体原因]

## 时间线
[从告警到现在的关键事件]

## 修复建议
[具体的修复步骤]

## 预防措施
[如何避免同类问题]
PROMPT
```

### 实际排查示例

```bash
# 当收到告警时，一行命令开始排查
claude code --prompt "$(cat diagnose-prompt.md)" --service api-gateway --time-range "15min"

# Agent 会自动执行以下操作：
# 1. curl health endpoint → 503
# 2. tail error.log → 发现大量 connection timeout
# 3. kubectl get pods → 发现 2/5 个 pod 处于 CrashLoopBackOff
# 4. kubectl logs → 看到 OOMKilled
# 5. 综合分析 → 内存泄漏导致 OOM
```

Agent 的输出：

```
## 故障概要
api-gateway 服务因内存泄漏导致 2/5 个 pod 被 OOM Kill

## 根因分析
- kubectl logs 显示 /api/v1/export 接口在处理大文件时内存占用持续增长
- jvm heap 从正常的 512MB 在 30 分钟内增长到 1.2GB
- OOM 发生前 30 秒，该接口的请求量突增 5 倍

## 时间线
- 10:15  /api/v1/export 接口请求量开始上升
- 10:35  pod-2 OOMKilled
- 10:38  pod-5 OOMKilled
- 10:39  告警触发
- 10:40  开始排查

## 修复建议
1. 立即：kubectl scale deployment api-gateway --replicas=8（增加副本分散压力）
2. 短期：在 /api/v1/export 增加限流（rate limit 100 req/min）
3. 长期：修复 export 接口的内存泄漏，增加完整测试

## 预防措施
- 添加接口级别的内存监控告警
- 为 /api/v1/export 添加单机并发数限制
```

---

## 效果

在 4 个微服务组成的生产环境中使用一个月：

| 指标 | 变化 |
|------|------|
| 平均故障排查时间 (MTTR) | 35min → 12min |
| 一线值班可处理的问题比例 | 40% → 75% |
| 需要升级到二线的事件 | 减少 55% |
| 凌晨被叫醒后的恢复时间 | 45min → 15min |

**典型场景数据**：

| 场景 | 人工排查 | Agent 排查 |
|------|----------|------------|
| OOM 排查 | 20min | 3min |
| 503 错误率突增 | 30min | 5min |
| 数据库连接池耗尽 | 45min | 8min |
| 证书过期 | 15min | 2min |

---

## 要点

1. **白名单命令是关键** — Shell MCP 非常强大，但也非常危险。必须通过 `ALLOWED_COMMANDS` 严格限制可执行的命令，绝不能让 Agent 能跑 `rm`、`sudo`、`dd`。

2. **诊断流程要固化** — 排查路径是有套路的（先看健康检查 → 再看错误日志 → 再看监控）。把套路写成 prompt，Agent 就不容易跑偏。

3. **输出结构化** — 故障报告要有固定格式。这样不管是人看还是后续自动处理，都能快速找到关键信息。

4. **坑：日志量太大** — `tail -n 1000` 可能产生几千行输出，超过 Claude 的上下文限制。建议先 `tail -n 100`，如果不够再加。或者在 prompt 里预置"先取 100 行，不够再取更多"的策略。

5. **只读原则** — 这个版本的 Agent 只能是只读诊断。如果要自动修复，需要额外加一层人工确认（比如 Agent 给出命令，人确认后再执行）。

---

## 扩展思路

- 对接告警系统：PagerDuty/飞书 Webhook 自动拉起诊断流程
- **自修复模式**（高风险）：对于已知模式（证书过期、磁盘写满），Agent 可以直接执行修复命令
- 诊断报告自动同步到事后复盘文档
- 集成 On-Call 轮值表：自动 @ 当前值班人

---

## 相关资源

- [Shell MCP Server](https://github.com/modelcontextprotocol/servers/tree/main/src/shell)
- [Claude Code MCP 配置](https://docs.anthropic.com/en/docs/claude-code/mcp)
- [Google SRE 手册 — 故障排查方法论](https://sre.google/sre-book/)
