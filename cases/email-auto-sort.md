---
title: 用 n8n + LLM 搭建邮件智能分类与自动回复 Agent
author: "@potato"
date: 2026-06
stack: n8n, GPT-4o-mini, IMAP, Gmail API, Airtable
difficulty: 入门
---

## 场景

销售/运营同学每天收到 100+ 封邮件，其中：

- 30% 是垃圾/推广邮件
- 40% 是可标准回复的常见问题（询价、合作邀约、发票申请）
- 20% 是需要内部流转的（技术支持、售后投诉）
- 10% 才是真正需要人工仔细处理的

人工一封封看和分发的成本极高，且容易漏掉紧急邮件。

目标是：**让 Agent 自动分类邮件，对常见问题自动回复，复杂邮件标记优先级并推送给对应负责人**。

---

## 实现

### 架构

```
新邮件到达 → n8n 触发 → LLM 分类 → ① 垃圾邮件 → 归档
                          → ② 标准问题 → 模板回复
                          → ③ 内部流转 → 创建工单
                          → ④ 紧急重要 → 飞书/Slack 高优告警
```

### 核心配置

**n8n 工作流节点配置**

```json
{
  "nodes": [
    {
      "type": "n8n-nodes-base.emailReadImap",
      "name": "监听新邮件",
      "parameters": {
        "mailbox": "INBOX",
        "format": "simple"
      }
    },
    {
      "type": "n8n-nodes-base.openAi",
      "name": "LLM 分类与意图识别",
      "parameters": {
        "model": "gpt-4o-mini",
        "prompt": "请分析以下邮件，输出 JSON 格式的分类结果：\n\n发件人：{{$json.from}}\n主题：{{$json.subject}}\n正文：{{$json.text}}\n\n要求：\n1. category: spam / inquiry / partnership / support / urgent / other\n2. priority: P0（紧急）/ P1（重要）/ P2（普通）/ P3（低优）\n3. sentiment: positive / neutral / negative\n4. suggested_action: auto_reply / forward_to_sales / forward_to_support / manual_review\n5. reply_draft: 如果建议自动回复，请生成一封专业、简洁的中文回复草稿\n6. summary: 邮件一句话摘要"
      }
    }
  ]
}
```

**自动回复模板策略**

| 邮件类型 | 自动回复模板 | 触发条件 |
|----------|--------------|----------|
| 产品询价 | "感谢询价，我们的产品定价方案见附件..." | 关键词"价格"/"报价"/"多少钱" |
| 合作邀约 | "感谢关注，请将贵司介绍和合作方案发送至..." | 关键词"合作"/"商务" |
| 发票申请 | "已收到发票申请，财务将在 3 个工作日内处理..." | 关键词"发票"/"开票" |
| 售后问题 | "已收到您的反馈，技术支持将在 2 小时内联系您..." | 关键词" bug"/"问题"/"故障" |

**工单流转规则**

```python
# n8n Function 节点
const result = JSON.parse($json.content);

if (result.category === "support") {
  return [{ json: { assignee: "技术支持组", label: "售后工单", sla: "2h" } }];
} else if (result.category === "partnership") {
  return [{ json: { assignee: "商务BD组", label: "合作邀约", sla: "24h" } }];
} else if (result.priority === "P0") {
  return [{ json: { assignee: "值班负责人", label: "紧急", sla: "30min" } }];
}
```

**飞书高优告警卡片**

```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": {
        "tag": "plain_text",
        "content": "🚨 紧急邮件需处理"
      },
      "template": "red"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**发件人：** {{$json.from}}\n**主题：** {{$json.subject}}\n**摘要：** {{$json.summary}}\n**情感：** {{$json.sentiment}}"
        }
      },
      {
        "tag": "action",
        "actions": [
          { "tag": "button", "text": "查看邮件", "url": "{{$json.webLink}}" },
          { "tag": "button", "text": "标记已处理", "type": "primary" }
        ]
      }
    ]
  }
}
```

---

## 效果

在 30 人左右 SaaS 团队场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 日均处理邮件量 | 150+ 封 |
| 自动回复占比 | 42%（标准问题无需人工） |
| 紧急邮件漏掉率 | 约 8% → 0% |
| 邮件首次响应时间 | 平均 4h → 平均 5min（自动回复） |
| 人工邮件处理时间 | 2h/天 → 40min/天 |
| 分类准确率 | 89% |

**翻车案例**：Agent 把一封带有"免费试用"字样的客户投诉误分为推广邮件归档了。后来在 prompt 中加了规则：如果发件人域名是客户列表中的，优先走人工审核。

---

## 要点

1. **发件人白名单优先** — 已知客户、合作伙伴的邮件不应该被自动归档，哪怕内容像垃圾邮件。维护一个白名单域名/邮箱列表是保险措施。

2. **自动回复要有"不是真人"的提示** — 在自动回复末尾加上"此邮件由智能助手自动回复，如需人工服务请回复'人工'"，避免用户误以为邮件无人处理。

3. **P0 邮件宁可误报不要漏报** — 客户投诉、服务器告警、监管通知这类邮件漏掉的代价极大。分类 prompt 中要对"负面情感"和"紧急关键词"做高敏感度设置。

4. **定期 review LLM 分类结果** — 每周抽 20 封让 LLM 分类并人工核对，把分类错误的案例加入 few-shot prompt 中持续优化。

---

## 扩展思路

- 接入 CRM 系统，自动把邮件内容同步到客户联系记录
- 多语言支持：英文询价自动用英文回复，日文邮件自动翻译后分类
- 邮件聚类：把同一主题的往来邮件串成线程，方便查看上下文
- 发件人画像：基于历史邮件自动生成对方公司/角色的背景信息

---

## 相关资源

- [n8n 官方文档](https://docs.n8n.io)
- [n8n Email 节点](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.emailreadimap/)
- [飞书消息卡片构建工具](https://open.feishu.cn/tool/cardbuilder)
- [Gmail API 快速入门](https://developers.google.com/gmail/api/quickstart/python)
