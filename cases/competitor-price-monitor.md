---
title: 用 Python + Playwright 搭建竞品价格监控与策略分析 Agent
author: "@potato"
date: 2026-06
stack: Python, Playwright, GPT-4o-mini, PostgreSQL, 钉钉
difficulty: 进阶
---

## 场景

电商运营团队每天需要监控竞品的价格变动、促销活动和库存状态。传统做法是由运营人员手动打开竞品店铺页面记录，痛点在于：

1. **频率低**：人力有限，通常只能抽查 10-20 个 SKU，无法全量覆盖
2. **反应慢**：竞品凌晨调价或上活动，运营次日上班才发现，错过跟价窗口
3. **信息碎片化**：价格、主图、促销文案、库存状态分散在不同位置，人工汇总效率低
4. **策略分析靠经验**：发现竞品降价后，判断是"清仓"还是"抢流量"依赖运营个人经验

该 Agent 目标：**每小时自动抓取指定竞品 SKU 的价格、促销信息、库存状态，异常变动时即时推送钉钉，并给出策略建议**。

---

## 实现

### 整体架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   定时调度       │────▶│  Playwright 爬虫 │────▶│  结构化解析     │
│  (APScheduler)  │     │  (无头浏览器)     │     │  (LLM/规则)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                    ┌────────────────────────────────────▼────────────────┐
                    │  价格对比引擎 → 异常检测 → 策略分析Agent → 钉钉推送   │
                    └──────────────────────────────────────────────────────┘
```

### 核心配置

**1. Playwright 爬虫（反爬友好）**

```python
# crawler.py
from playwright.sync_api import sync_playwright
import random

USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...",
    # 多准备几个真实 UA
]

def fetch_product(url: str) -> dict:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        context = browser.new_context(
            user_agent=random.choice(USER_AGENTS),
            viewport={"width": 1920, "height": 1080}
        )
        page = context.new_page()
        page.goto(url, wait_until="networkidle")
        
        # 等待页面关键元素加载
        page.wait_for_selector("[class*='price']", timeout=10000)
        
        # 提取原始 HTML 片段
        html_snippet = page.content()
        browser.close()
        return {"url": url, "html": html_snippet}
```

**2. LLM 结构化提取**

```python
# extractor.py
from openai import OpenAI
import json

client = OpenAI()

EXTRACTION_PROMPT = """从以下电商商品页面 HTML 中提取结构化信息，输出 JSON：
{
  "current_price": float,
  "original_price": float,
  "promotion_tags": ["满减", "限时折扣", ...],
  "stock_status": "有货/缺货/预售",
  "main_selling_points": ["..."],
  "gift_info": "..."
}

HTML 片段：
{html}

注意：
1. 价格只取数字，不带货币符号
2. 促销标签去重，最多 5 个
3. 如果信息缺失，字段值为 null"""

def extract(html: str) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": EXTRACTION_PROMPT.format(html=html[:8000])}],
        response_format={"type": "json_object"}
    )
    return json.loads(resp.choices[0].message.content)
```

**3. 价格异常检测与策略分析**

```python
# analyzer.py
from datetime import datetime, timedelta
import psycopg2

conn = psycopg2.connect("dbname=monitor user=postgres")

def detect_anomaly(sku_id: str, new_data: dict) -> dict:
    cursor = conn.cursor()
    cursor.execute(
        "SELECT current_price, promotion_tags, created_at FROM price_history "
        "WHERE sku_id = %s ORDER BY created_at DESC LIMIT 1",
        (sku_id,)
    )
    last = cursor.fetchone()
    
    alerts = []
    if last:
        price_change = (new_data["current_price"] - last[0]) / last[0]
        if price_change < -0.15:
            alerts.append({"type": "大幅降价", "change": f"{price_change:.1%}", "urgency": "高"})
        if last[1] != new_data.get("promotion_tags"):
            alerts.append({"type": "促销变更", "old": last[1], "new": new_data["promotion_tags"], "urgency": "中"})
    
    return {"alerts": alerts, "sku_id": sku_id}

def generate_strategy(sku_data: dict, competitor_data: dict) -> str:
    """用 LLM 分析竞品行为意图，给出应对建议"""
    prompt = f"""你是电商运营策略分析师。竞品数据如下：
竞品价格：{competitor_data['current_price']}，原价：{competitor_data['original_price']}
促销：{competitor_data.get('promotion_tags', [])}
库存：{competitor_data.get('stock_status')}

我方 SKU 定价：{sku_data['our_price']}

请分析竞品意图（清仓/抢流量/日常促销/大促预热），并给出 2-3 条具体应对建议。每条不超过 30 字。"""
    
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return resp.choices[0].message.content
```

**4. 定时任务与钉钉推送**

```python
# scheduler.py
from apscheduler.schedulers.blocking import BlockingScheduler
from dingtalkchatbot.chatbot import DingtalkChatbot

webhook = "https://oapi.dingtalk.com/robot/send?access_token=XXX"
robot = DingtalkChatbot(webhook)

def hourly_job():
    for sku in WATCH_LIST:
        raw = fetch_product(sku["competitor_url"])
        extracted = extract(raw["html"])
        anomaly = detect_anomaly(sku["id"], extracted)
        
        if anomaly["alerts"]:
            strategy = generate_strategy(sku, extracted)
            robot.send_markdown(
                title="竞品价格异动",
                text=f"### 🚨 竞品异动：{sku['name']}\n"
                     f"- 价格：{extracted['current_price']}\n"
                     f"- 变动：{anomaly['alerts'][0]['change']}\n"
                     f"- 策略建议：\n{strategy}"
            )
        
        # 存入历史
        save_to_db(sku["id"], extracted)

scheduler = BlockingScheduler()
scheduler.add_job(hourly_job, "interval", hours=1)
scheduler.start()
```

---

## 效果

在 SKU 300+、竞品店铺 20 家的电商团队中，预期效果：

| 指标 | 变化 |
|------|------|
| 监控覆盖 SKU 数 | 20 个 → 全部 300 个 |
| 价格变动发现延迟 | 次日 → 1 小时内 |
| 运营每日手动监控时间 | 2h → 0（仅看推送） |
| 跟价决策时间 | 平均 4h（收集信息） → 10min |
| 促销遗漏率 | 约 15%（人工漏看） → 5% |

---

## 要点

1. **反爬是核心挑战** — 头部电商平台的反爬策略很强。建议：控制请求频率（每小时一次，每次间隔 5-10 秒）、使用住宅代理 IP、模拟真实用户行为（滚动、停留）。不要暴力爬取。

2. **页面结构会变** — 电商前端经常改版，XPath/CSS 选择器会失效。建议用 LLM 从 HTML 中自由提取，而非硬编码选择器，容错性更高。

3. **价格口径要统一** — 注意区分"到手价"（满减后）和"标价"。建议爬取时同时抓取标价、促销规则，自己计算到手价，避免误判。

4. **Legal 边界** — 爬取公开页面通常合法，但不要登录账号爬取内部数据。建议只监控公开店铺页，且频率不要过于激进。

---

## 延伸阅读

- [Playwright 官方文档](https://playwright.dev/python/)
- [APScheduler 使用指南](https://apscheduler.readthedocs.io/)
- [钉钉机器人开发文档](https://open.dingtalk.com/document/isv/server-api/send-a-robot-group-message)

---

*Last updated: 2025-06-02*
