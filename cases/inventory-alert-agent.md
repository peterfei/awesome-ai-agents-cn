---
title: 用 Python + LLM 搭建库存预警与智能补货建议 Agent
author: "@potato"
date: 2026-06
stack: Python, Pandas, GPT-4o-mini, PostgreSQL, 钉钉/飞书
difficulty: 进阶
---

## 场景

电商/零售的库存管理是门平衡的艺术：

1. **缺货 = 少卖** — 热销品断货，流量白白浪费
2. **压货 = 资金占用** — 滞销品积压在仓，仓储费越来越高
3. **补货节奏靠人猜** — 采购凭经验拍脑袋，旺季补晚了，淡季补多了

ERP 系统虽然能看当前库存，但缺少"未来趋势判断"和"补货建议生成"。

目标是：**Agent 每天自动分析销售趋势、预测断货风险、生成补货建议并推送给采购**。

---

## 实现

### 架构

```
销售数据 + 库存数据 → 趋势计算 → 断货预测 → LLM 生成补货建议 → 钉钉推送采购
         ↑                                              ↓
    季节/大促/市场数据                               自动生成采购单草稿
```

### 核心实现

**第一步：数据聚合与指标计算**

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://...")

# 拉取近 90 天销售数据
sales = pd.read_sql("""
    SELECT sku, date, qty_sold, channel, promotion_flag
    FROM daily_sales
    WHERE date >= CURRENT_DATE - INTERVAL '90 days'
""", engine)

# 当前库存
inventory = pd.read_sql("""
    SELECT sku, warehouse, qty_on_hand, qty_inbound, safety_stock
    FROM inventory_snapshot
    WHERE snapshot_date = CURRENT_DATE
""", engine)

# 计算关键指标
def calculate_metrics(sales_df, inventory_df):
    """计算每个 SKU 的核心库存指标"""
    
    results = []
    for sku in inventory_df["sku"].unique():
        sku_sales = sales_df[sales_df["sku"] == sku]
        sku_inv = inventory_df[inventory_df["sku"] == sku]
        
        # 日均销量（近 7 天和近 30 天加权）
        d7_avg = sku_sales[sku_sales["date"] >= pd.Timestamp.now() - pd.Timedelta(days=7)]["qty_sold"].mean()
        d30_avg = sku_sales["qty_sold"].mean()
        daily_avg = d7_avg * 0.6 + d30_avg * 0.4  # 近期权重更高
        
        # 当前总库存（在库 + 在途）
        total_stock = sku_inv["qty_on_hand"].sum() + sku_inv["qty_inbound"].sum()
        
        # 可售天数
        days_remaining = total_stock / daily_avg if daily_avg > 0 else float('inf')
        
        # 销售趋势（近 7 天 vs 前 7 天）
        recent = sku_sales[sku_sales["date"] >= pd.Timestamp.now() - pd.Timedelta(days=7)]["qty_sold"].sum()
        previous = sku_sales[(sku_sales["date"] >= pd.Timestamp.now() - pd.Timedelta(days=14)) & 
                            (sku_sales["date"] < pd.Timestamp.now() - pd.Timedelta(days=7))]["qty_sold"].sum()
        trend = (recent - previous) / previous if previous > 0 else 0
        
        # 大促系数（近期是否有 promotion）
        promo_boost = 1.3 if sku_sales["promotion_flag"].any() else 1.0
        
        results.append({
            "sku": sku,
            "daily_avg": daily_avg,
            "total_stock": total_stock,
            "days_remaining": days_remaining,
            "trend": trend,
            "promo_boost": promo_boost,
            "safety_stock": sku_inv["safety_stock"].sum()
        })
    
    return pd.DataFrame(results)

metrics = calculate_metrics(sales, inventory)
```

**第二步：风险分级**

```python
def classify_risk(row):
    """根据可售天数和趋势判断风险等级"""
    
    days = row["days_remaining"]
    trend = row["trend"]
    safety = row["safety_stock"]
    
    # 考虑趋势加速消耗
    effective_days = days / (1 + max(trend, 0))
    
    if effective_days <= 7:
        return "🔴 紧急", "7 天内断货，需立即补货"
    elif effective_days <= 14:
        return "🟠 预警", "14 天内可能断货，建议本周下单"
    elif effective_days <= 30:
        return "🟡 关注", "库存偏低，关注销量变化"
    elif days > 90 and trend < -0.3:
        return "🔵 滞销", "动销下降，考虑促销清货"
    else:
        return "🟢 正常", None

metrics["risk_level"], metrics["risk_reason"] = zip(*metrics.apply(classify_risk, axis=1))
```

**第三步：LLM 生成补货建议**

```python
from openai import OpenAI

client = OpenAI()

def generate_restock_advice(sku_data):
    """用 LLM 结合业务知识生成采购建议"""
    
    prompt = f"""
    你是一个资深供应链分析师。请为以下商品生成补货建议。

    ## 商品信息
    - SKU：{sku_data['sku']}
    - 当前总库存：{sku_data['total_stock']} 件
    - 日均销量：{sku_data['daily_avg']:.1f} 件
    - 可售天数：{sku_data['days_remaining']:.1f} 天
    - 销售趋势：近两周环比 {'+' if sku_data['trend'] > 0 else ''}{sku_data['trend']*100:.0f}%
    - 安全库存线：{sku_data['safety_stock']} 件
    - 大促系数：{sku_data['promo_boost']}

    ## 请输出
    1. 建议采购数量（含计算逻辑）
    2. 建议下单时间
    3. 风险提醒（如趋势加速需多备）
    4. 是否建议调整安全库存线

    规则：
    - 供应商交期按 14 天计算
    - 建议库存覆盖 = 交期 + 安全周期(7天) + 大促缓冲
    - 单次采购量不超过供应商起订量的 3 倍
    """
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    
    return response.choices[0].message.content

# 只对风险 SKU 生成建议
risk_skus = metrics[metrics["risk_level"].str.contains("🔴|🟠|🔵")]
advice = {}
for _, row in risk_skus.iterrows():
    advice[row["sku"]] = generate_restock_advice(row)
```

**第四步：钉钉推送与采购单草稿**

```python
def send_alert(sku, level, reason, advice_text):
    """发送钉钉卡片消息给采购团队"""
    
    if "🔴" in level:
        at_mobiles = ["采购主管手机号", "运营负责人手机号"]  # 紧急@人
    else:
        at_mobiles = []
    
    message = {
        "msgtype": "markdown",
        "markdown": {
            "title": f"库存预警：{sku}",
            "text": f"""
## {level} · {sku}

{reason}

### 补货建议
{advice_text}

### 快捷操作
- [查看销量趋势]
- [生成采购单草稿]
- [调整安全库存]
            """
        },
        "at": {
            "atMobiles": at_mobiles,
            "isAtAll": False
        }
    }
    
    requests.post(
        "https://oapi.dingtalk.com/robot/send?access_token=...",
        json=message
    )

# 自动生成采购单草稿（对接 ERP）
def generate_purchase_draft(sku, suggested_qty):
    return {
        "sku": sku,
        "qty": suggested_qty,
        "supplier": get_default_supplier(sku),
        "expected_delivery": (pd.Timestamp.now() + pd.Timedelta(days=14)).strftime("%Y-%m-%d"),
        "status": "draft",
        "source": "AI_agent"
    }
```

---

## 效果

在 SKU 2000+ 的电商场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 缺货率 | 8.5% → 3.2% |
| 滞销 SKU 占比 | 15% → 9%（提前识别并促销） |
| 采购决策时间 | 平均 2 天 → 平均 2 小时 |
| 安全库存周转天数 | 45 天 → 32 天 |
| 大促备货准确率 | 人工预估偏差 ±40% → AI 建议偏差 ±15% |
| 每日预警覆盖 SKU 数 | 全部 2000+ SKU 自动扫描 |

---

## 要点

1. **预测比规则更重要** — 固定安全库存（比如"低于 100 就补"）在销量波动时失灵。Agent 的核心价值是结合趋势动态调整建议，而不是简单阈值报警。

2. **季节性必须纳入模型** — 纯看近期销量会误判（比如夏季判断羽绒服需求为零，但秋冬需要提前备货）。建议接入品类季节性系数和历史同期数据。

3. **供应商交期是瓶颈** — 补货建议算得再准，供应商交期 30 天也救不了急。建议维护供应商实时交期数据，Agent 自动把交期长的 SKU 提前量放大。

4. **人工最终拍板不可少** — AI 建议采购 3000 件，但采购可能知道"供应商正在换产线，交期要翻倍"这类上下文。Agent 负责"给建议+给数据"，决策权留给采购。

---

## 扩展思路

- 接入天气 API：雨伞、空调等天气敏感 SKU 根据天气预报调整建议
- 竞品价格监控：竞品降价时，Agent 建议加速促销清库存而非补货
- 多仓调拨建议：不是每次都向供应商采购，优先建议 A 仓调货到 B 仓
- 供应链金融联动：大额采购建议自动计算资金占用和 ROI

---

## 相关资源

- [Pandas 时间序列分析](https://pandas.pydata.org/docs/user_guide/timeseries.html)
- [库存管理经典公式：经济订货量 EOQ](https://en.wikipedia.org/wiki/Economic_order_quantity)
- [钉钉群机器人 API](https://open.dingtalk.com/document/isvapp-server/create-an-robot)
- [PostgreSQL 窗口函数做滑动平均](https://www.postgresql.org/docs/current/tutorial-window.html)
