---
title: 用 LLM + BI 数据搭建数据异常监控与根因分析 Agent
author: "@potato"
date: 2026-06
stack: Python, SQL, GPT-4o, Pandas, Streamlit, 钉钉
difficulty: 进阶
---

## 场景

数据分析师/运营每天早上打开 BI 看板，需要快速回答："昨天 GMV 为什么跌了 15%？" 传统排查流程是：

1. 先看大盘，确认是整体跌还是某个渠道跌
2. 拆解维度：按平台、品类、新老客、投放渠道拆分
3. 对比上周同期，排除周期性因素
4. 交叉验证：是不是某个大促结束？是不是投放预算削减？

这个过程熟练的分析师也要 20-30 分钟，且依赖经验。非分析背景的运营常被 @ 后一脸茫然。当异常发生时（凌晨 2 点系统报警），往往没人能及时分析。

该 Agent 目标：**当核心指标偏离正常区间时，自动执行多维下钻分析，生成带归因结论的简报，推送给相关同学**。

---

## 实现

### 整体架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  指标监控层      │────▶│  自动下钻分析    │────▶│  LLM 归因总结    │
│  (SQL/BI API)   │     │  (Pandas/规则)   │     │  (生成自然语言)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  钉钉/飞书推送     │
                    │  (带下钻图表链接)  │
                    └─────────────────────┘
```

### 核心配置

**1. 指标异常检测**

```python
# anomaly_detector.py
import pandas as pd
from datetime import datetime, timedelta

def fetch_metric(metric_name: str, days: int = 30) -> pd.DataFrame:
    """从数仓拉取指标历史数据"""
    query = f"""
    SELECT date, {metric_name}, platform, category, user_type
    FROM daily_metrics
    WHERE date >= CURRENT_DATE - INTERVAL '{days} days'
    """
    return pd.read_sql(query, engine)

def detect_anomaly(df: pd.DataFrame, threshold_std: float = 2.0) -> dict:
    """基于历史均值和标准差检测异常"""
    df["date"] = pd.to_datetime(df["date"])
    df = df.sort_values("date")
    
    # 计算 7 日移动平均和标准差
    df["rolling_mean"] = df["gmv"].rolling(window=7, min_periods=3).mean()
    df["rolling_std"] = df["gmv"].rolling(window=7, min_periods=3).std()
    
    latest = df.iloc[-1]
    deviation = (latest["gmv"] - latest["rolling_mean"]) / latest["rolling_std"]
    
    if abs(deviation) > threshold_std:
        return {
            "is_anomaly": True,
            "metric": "gmv",
            "current": latest["gmv"],
            "expected": latest["rolling_mean"],
            "deviation_std": deviation,
            "direction": "up" if deviation > 0 else "down"
        }
    return {"is_anomaly": False}
```

**2. 自动多维下钻**

```python
# drill_down.py

def drill_down(df: pd.DataFrame, anomaly_date: str) -> dict:
    """按多个维度拆解，找出贡献度最大的异常维度"""
    
    # 基准：前 7 天均值
    base = df[df["date"] < anomaly_date].groupby("platform")["gmv"].mean()
    current = df[df["date"] == anomaly_date].groupby("platform")["gmv"].sum()
    
    platform_impact = ((current - base) / base).sort_values(ascending=False)
    
    # 找出贡献最大的维度
    top_platform = platform_impact.abs().idxmax()
    
    # 在该平台内继续下钻品类
    platform_df = df[df["platform"] == top_platform]
    base_cat = platform_df[platform_df["date"] < anomaly_date].groupby("category")["gmv"].mean()
    current_cat = platform_df[platform_df["date"] == anomaly_date].groupby("category")["gmv"].sum()
    cat_impact = ((current_cat - base_cat) / base_cat).sort_values(ascending=False)
    
    return {
        "primary_dimension": "platform",
        "primary_value": top_platform,
        "primary_impact": platform_impact[top_platform],
        "secondary_dimension": "category",
        "secondary_impact": cat_impact.head(3).to_dict(),
        "base_value": base.sum(),
        "current_value": current.sum()
    }
```

**3. LLM 归因总结**

```python
# root_cause_llm.py
from openai import OpenAI

client = OpenAI()

def generate_insight(anomaly: dict, drill_down: dict, context: str = "") -> str:
    prompt = f"""你是一名资深数据分析师。以下是一次数据异常的自动分析结果，请用 3-5 句话向运营同学解释原因。

异常指标：{anomaly['metric']}
当前值：{anomaly['current']:.0f}（预期 {anomaly['expected']:.0f}）
偏离度：{anomaly['deviation_std']:.1f} 个标准差

下钻结果：
- 主要异常维度：{drill_down['primary_dimension']} = {drill_down['primary_value']}
- 该平台/渠道的影响：{drill_down['primary_impact']:.1%}
- 子维度 TOP3 变化：{drill_down['secondary_impact']}

附加信息：{context}

要求：
1. 用中文，非技术语言
2. 先给结论，再给证据
3. 如果是下跌，给出最可能的原因猜测
4. 如果无法确定原因，诚实说"需要进一步排查""""

    resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3
    )
    return resp.choices[0].message.content
```

**4. 定时巡检与推送**

```python
# scheduler.py
from apscheduler.schedulers.blocking import BlockingScheduler

def daily_check():
    df = fetch_metric("gmv", days=30)
    anomaly = detect_anomaly(df)
    
    if anomaly["is_anomaly"]:
        drill = drill_down(df, anomaly_date=str(datetime.now().date()))
        insight = generate_insight(anomaly, drill)
        
        send_dingtalk(
            title=f"🚨 数据异常：GMV {anomaly['direction']} {abs(anomaly['deviation_std']):.1f}σ",
            content=insight + "\n\n[点击查看下钻明细](https://bi.company.com/dashboard/anomaly)"
        )

scheduler = BlockingScheduler()
# 每天早上 9 点巡检
scheduler.add_job(daily_check, "cron", hour=9, minute=0)
scheduler.start()
```

---

## 效果

在日均 GMV 500 万+的电商业务中，预期效果：

| 指标 | 变化 |
|------|------|
| 异常发现时间 | 人工看板 2-3h → 自动 9:00 推送 |
| 根因定位时间 | 分析师 20-30min → Agent 30s 生成初判 |
| 运营被 @ 问"为什么跌了"的频率 | 每天 5-10 次 → 降至 1-2 次（仅复杂异常） |
| 分析师重复排查时间 | 每天 1.5h → 0.3h |
| 凌晨异常覆盖 | 无覆盖 → 自动报警（如设定 2:00 巡检） |

---

## 要点

1. **基线选择决定敏感度** — 用"前 7 天均值"做基线在周末/大促后会误判。建议：剔除大促日期、用去年同期对比、区分工作日/周末模式。

2. **下钻维度要业务化** — 纯技术维度（按数据库表字段拆分）没有意义。下钻维度必须是业务关心的：新老客、渠道、品类、活动/非活动。

3. **LLM 归因是"猜测"不是"事实"** — Agent 能指出"天猫渠道下跌 30%，可能是大促结束后的自然回落"，但真正的根因（如某投放计划被误停）需要人工确认。建议在推送中标注"推测原因，请运营复核"。

4. **避免噪音** — 如果指标本身波动就大（如小时级 GMV），不要设太敏感的阈值，否则天天报警会被忽略。建议先跑一周观察误报率，再调整 threshold_std。

---

## 延伸阅读

- [Pandas GroupBy 官方文档](https://pandas.pydata.org/docs/user_guide/groupby.html)
- [APScheduler 定时任务](https://apscheduler.readthedocs.io/)
- [数据异常检测方法综述](https://towardsdatascience.com/anomaly-detection-methods-in-python-3f0c70e760f9)

---

*Last updated: 2025-06-02*
