# 用 Dify + LLM 搭建智能售后回访 Agent

> 标签：`客服` `售后` `Dify` `RAG` `GPT-4o-mini`

---

## 场景

某家电品牌日均发货 3000 单，售后回访原由人工客服完成，存在以下痛点：

- **覆盖率不足**：只能覆盖 20% 的高客单价订单，中低价款几乎无回访
- **体验千篇一律**：固定话术模板，用户觉得"像机器人"
- **差评预警滞后**：用户的不满往往在微信/小红书爆发后才被品牌发现

该 Agent 目标：**在订单签收后 7 天自动触发多渠道回访，收集满意度并识别差评风险，实时同步给客服主管**。

---

## 实现

### 整体架构

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   订单系统    │────▶│  Dify Agent   │────▶│  用户触达层   │
│  (ERP/OMS)   │     │  (RAG+LLM)   │     │  短信/企微/AI电话│
└──────────────┘     └──────────────┘     └──────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  结果聚合与风险预警  │
                    │  (飞书多维表格/钉钉) │
                    └─────────────────────┘
```

### 核心配置

**1. Dify 工作流设计**

- **触发节点**：每天凌晨从 ERP 拉取"签收日期 = 7 天前"的订单
- **RAG 检索**：根据商品 SKU 召回该品类的常见故障知识、安装注意事项、使用技巧
- **LLM 生成话术**：基于用户画像（性别、地区、购买历史）生成个性化回访文案
- **评分卡**：对用户的文字/语音回复做情感分析，输出 NPS 分数与风险标签

**2. Prompt 设计（RAG + 个性化）**

```yaml
# Dify Prompt 片段
system_prompt: |
  你是一位温暖耐心的家电售后客服助手。根据以下商品信息和用户画像，生成一段不超过 80 字的回访开场白。
  要求：
  1. 提及具体商品名称，不使用"亲""宝贝"等淘系用语
  2. 语气像朋友关心，而非问卷调查
  3. 针对该商品高频使用场景提问

context: |
  {{#context#}}
  商品：{{product_name}}
  品类知识：{{retrieved_knowledge}}
  用户画像：{{user_profile}}
```

**3. 情感分析与风险分级**

```python
# sentiment_classifier.py
from openai import OpenAI

client = OpenAI()

def classify_feedback(user_reply: str) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "你是一个用户反馈分析专家。请对用户回访回复进行情感分析，输出JSON格式：{\"nps\": 0-10, \"risk\": \"无/低/中/高\", \"keywords\": [\"...\"], \"summary\": \"20字以内\"}"},
            {"role": "user", "content": user_reply}
        ],
        response_format={"type": "json_object"}
    )
    return json.loads(resp.choices[0].message.content)
```

**4. 差评预警与工单流转**

```python
# alert_flow.py
from feishu_api import send_message

def handle_risk(order_id: str, risk_level: str, summary: str):
    if risk_level == "高":
        send_message(
            chat_id="售后主管群",
            content=f"🚨 差评风险预警\n订单：{order_id}\n摘要：{summary}\n建议：立即人工介入"
        )
        create_crm_ticket(order_id, priority="urgent")
    elif risk_level == "中":
        send_message(
            chat_id="售后跟进群",
            content=f"⚠️ 潜在不满\n订单：{order_id}\n摘要：{summary}"
        )
```

---

## 效果

- **覆盖率提升**：从 20% 提升到 **95%**（除拒访用户外全部覆盖）
- **NPS 回收率**：用户愿意回复的比例从 8% 提升到 **22%**，因话术更自然
- **差评拦截**：预期可识别并安抚大部分高不满用户，减少社交媒体差评曝光
- **人工释放**：客服团队日均回访工时从 6 人时降至 **1 人时**（仅处理高风险工单）

---

## 关键要点

- **RAG 决定专业度**：接入商品说明书和故障 FAQ 后，用户感知明显更专业，回访成功率更高
- **风险分级避免噪音**：不要把所有"一般"反馈都推给主管，只有"高"风险才需要人工介入
- **多渠道互补**：短信用于触达，企微用于深度对话，AI 电话用于高客单价用户，效果最佳

---

## 延伸阅读

- [Dify 工作流编排文档](https://docs.dify.ai/guides/workflow)
- [情感分析最佳实践](https://platform.openai.com/docs/guides/prompt-engineering)
- [企业微信客服 API](https://developer.work.weixin.qq.com/document/path/90236)

---

*Last updated: 2025-06-02*
