---
title: 用 n8n + GPT 做销售线索自动筛选与跟进 Agent
author: "@potato"
date: 2026-06
stack: n8n, GPT-4o-mini, HubSpot API, 企业微信, Python
difficulty: 入门
---

## 场景

B2B SaaS 公司每月从官网、展会、内容营销等渠道获取 2000+ 条线索，但销售团队只有 10 人，人均每月要处理 200 条。问题在于：

1. **线索质量参差不齐** — 大量学生、竞品调研、无效邮箱
2. **跟进不及时** — 线索分配靠手动，优质线索可能过了 48 小时才有人联系
3. **打标签不规范** — 每个销售对线索的标注标准不同，难以统计

目标是：**让 Agent 自动评估线索质量、打分定级、分配销售并触发首轮触达**。

---

## 实现

### 架构

```
线索入库 → 数据补全（企查查/LinkedIn）→ LLM 打分定级 → ① A级 → 立即分配 + 企微告警
                                      → ② B级 → 入 nurture 池 + 自动邮件
                                      → ③ C级 → 批量培育 / 暂不处理
                                      → ④ 垃圾 → 归档
```

### 核心配置

**第一步：线索数据补全**

```python
# n8n Function 节点
import requests

def enrich_lead(lead):
    # 通过邮箱域名查公司信息
    domain = lead["email"].split("@")[1]
    
    # 企查查 API（国内线索）
    qcc = requests.get(
        f"https://api.qcc.com/searches?key={domain}",
        headers={"Authorization": f"Token {env.QCC_TOKEN}"}
    )
    
    # 用 LLM 判断职位层级
    title = lead.get("job_title", "")
    
    return {
        **lead,
        "company_name": qcc.json().get("name"),
        "company_size": qcc.json().get("scale"),
        "industry": qcc.json().get("industry"),
    }
```

**第二步：LLM 打分定级**

```yaml
name: 线索评分助手
model: gpt-4o-mini
prompt: |
  你是一位资深的 B2B 销售运营。请根据以下线索信息，给出专业的评分和分级建议。

  ## 线索信息
  - 姓名：{{name}}
  - 邮箱：{{email}}
  - 公司：{{company_name}}
  - 公司规模：{{company_size}}
  - 职位：{{job_title}}
  - 行业：{{industry}}
  - 来源渠道：{{source}}（官网表单/线下展会/白皮书下载/伙伴推荐）
  - 行为轨迹：{{behavior}}（浏览了哪些页面、是否试用了产品）

  ## 评分标准

  请输出 JSON：
  {
    "score": 0-100,
    "grade": "A/B/C/D",
    "reason": "评分理由，一句话",
    "fit_icp": true/false,  // 是否符合理想客户画像
    "priority": "立即跟进/24h内/ nurture培育/ 放弃",
    "suggested_owner": "按行业匹配：互联网→张销售，制造业→李销售，教育→王销售",
    "first_touch_content": "建议的首轮触达话术"
  }

  评分规则：
  - 80-100 = A（决策人 + 匹配行业 + 主动试用产品）
  - 60-79 = B（相关人士 + 有需求信号）
  - 40-59 = C（信息不全或匹配度一般）
  - <40 = D（学生邮箱、竞品、明显不匹配）
```

**第三步：自动分配与触达**

```python
# n8n 中根据分级路由
grade = $json.grade

if grade == "A":
    # 立即分配 + 企微告警
    assignee = $json.suggested_owner
    send_wecom_alert(f"🔥 A级线索：{$json.company_name} {$json.name}，请立即跟进！")
    create_hubspot_task(assignee, $json, due="now")
elif grade == "B":
    # 自动发送培育邮件
    send_nurture_email($json.email, template="b_grade_nurture")
    create_hubspot_task($json.suggested_owner, $json, due="24h")
elif grade == "C":
    # 入培育池，加入邮件序列
    add_to_sequence($json.email, sequence="monthly_newsletter")
```

**第四步：企微销售告警卡片**

```
🔥 A级线索 · 立即跟进

公司：{{company_name}}（{{company_size}}人 · {{industry}}）
联系人：{{name}} · {{job_title}}
邮箱：{{email}}
来源：{{source}}
行为：{{behavior}}

🎯 建议话术：{{first_touch_content}}

[一键复制话术] [查看详情] [标记已联系]
```

---

## 效果

在年 ARR 500 万量级的 SaaS 场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 线索响应时间 | 平均 36h → A级线索 5min 内分配 |
| 销售人均月处理量 | 200 条 → 聚焦 50 条 A/B 级 |
| 线索→商机转化率 | 3.2% → 7.8% |
| 销售花在无效线索上的时间 | 40% → 12% |
| 首轮触达自动化率 | 65%（B/C 级自动邮件） |
| 评分一致性 | 人工打分方差大 → AI 标准统一 |

---

## 要点

1. **ICP（理想客户画像）要前置定义** — Agent 打分准不准，取决于你对"好客户"的定义是否清晰。建议在 prompt 中明确写出目标行业、公司规模、目标职位。

2. **不要完全自动化 A 级线索的触达** — 虽然可以自动发邮件/短信，但 B2B 高价值线索的第一次接触最好是真人电话。Agent 做到"秒分配"就够了，触达还是销售自己来。

3. **行为数据比静态信息更重要** — 一个"总监"头衔的人如果只是误点广告，不如一个"工程师"深度试用了产品。行为权重（试用、多次访问定价页）应该高于职位权重。

4. **定期回流打标结果** — 把实际成交客户的线索特征反馈给 Agent，用来优化评分 prompt。建议每月 review 一次"A 级但实际未成交"的线索，找出评分模型的盲区。

---

## 扩展思路

- 接入官网行为追踪（Mixpanel/神策），自动把" pricing 页停留 >30s"作为强意向信号
- 多轮培育序列：B 级线索自动触发 5 封邮件 nurture，每封根据是否打开/点击调整后续内容
- 竞品线索识别：根据邮箱域名、公司名自动识别竞品调研，降低分配优先级
- 成交预测：基于历史数据训练模型，预测每条线索 90 天内成交概率

---

## 相关资源

- [HubSpot CRM API](https://developers.hubspot.com/docs/api/overview)
- [n8n HubSpot 节点](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.hubspot/)
- [企查查开发者平台](https://www.qcc.com)
- [BANT 销售资格框架](https://www.salesforce.com/resources/articles/sales-process/bant/)
