# 用 Python + LLM 搭建 BI 报告自动生成 Agent

> 标签：`数据分析` `BI` `Python` `Pandas` `LangChain` `Streamlit`

---

## 场景

某快消品牌电商运营团队每周一需要产出《上周全渠道销售复盘报告》。数据来源包括：

- 天猫/京东后台导出的 Excel 报表
- 自建数仓（PostgreSQL）中的订单、库存、广告消耗数据
- 第三方数据平台（如生意参谋）的竞品监控 CSV

传统做法是 2 名数据分析师花 1 天时间做数据清洗、透视、写 PPT。该 Agent 目标：**周一早上 9 点自动推送一份带洞察结论的 Markdown/HTML 报告到钉钉群**。

---

## 实现

### 整体架构

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  数据源接入层    │────▶│  数据处理与归因   │────▶│  洞察生成与排版   │
│  (Excel/CSV/DB) │     │  (Pandas + SQL)  │     │  (LLM + Jinja2) │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                         │
                                    ┌────────────────────▼────────────────────┐
                                    │              钉钉机器人推送               │
                                    └─────────────────────────────────────────┘
```

### 核心配置

**1. 数据接入与标准化**

```python
# data_loader.py
import pandas as pd
from sqlalchemy import create_engine
from pathlib import Path

DATA_SOURCES = {
    "tmall": "raw/tmall_weekly.xlsx",
    "jd": "raw/jd_weekly.xlsx",
    "dw": "postgresql+psycopg2://user:pass@localhost:5432/dw",
}

def load_and_normalize(source_key: str) -> pd.DataFrame:
    """统一列名、日期格式、金额单位"""
    if source_key in ("tmall", "jd"):
        df = pd.read_excel(DATA_SOURCES[source_key])
    else:
        engine = create_engine(DATA_SOURCES[source_key])
        df = pd.read_sql("SELECT * FROM weekly_summary", engine)

    # 统一列名映射
    COLUMN_MAP = {
        "订单金额": "gmv", "支付金额": "gmv", "销售额": "gmv",
        "访客数": "uv", "浏览量": "pv",
        "订单量": "orders", "支付件数": "orders",
    }
    df.rename(columns=COLUMN_MAP, inplace=True, errors="ignore")
    df["date"] = pd.to_datetime(df["date"])
    return df
```

**2. 关键指标计算与异常检测**

```python
# metrics.py
import pandas as pd
from datetime import datetime, timedelta

def calc_week_over_week(df: pd.DataFrame) -> dict:
    """计算周环比，并标记异常波动"""
    this_week = df[df["date"] >= datetime.now() - timedelta(days=7)]
    last_week = df[(df["date"] >= datetime.now() - timedelta(days=14)) &
                   (df["date"] < datetime.now() - timedelta(days=7))]

    metrics = {}
    for col in ["gmv", "uv", "orders"]:
        tw = this_week[col].sum()
        lw = last_week[col].sum()
        wow = (tw - lw) / lw if lw else 0
        metrics[col] = {"value": tw, "wow": wow, "alert": abs(wow) > 0.2}
    return metrics
```

**3. LLM 洞察生成**

```python
# insight_generator.py
from langchain import OpenAI, PromptTemplate
from langchain.chains import LLMChain

INSIGHT_PROMPT = PromptTemplate(
    input_variables=["metrics_str", "top_products", "ad_spend_roi"],
    template="""你是一名资深电商数据分析师。请根据以下本周数据给出 3-5 条核心洞察与行动建议。

关键指标周环比：
{metrics_str}

TOP5 爆款及同比变化：
{top_products}

广告消耗与 ROI：
{ad_spend_roi}

要求：
1. 使用中文，每条 50 字以内
2. 指出具体业务机会或风险
3. 给出可执行的运营动作"""
)

llm = OpenAI(model_name="gpt-4o", temperature=0.3)
insight_chain = LLMChain(llm=llm, prompt=INSIGHT_PROMPT)

def generate_insights(metrics: dict, products: str, ads: str) -> str:
    return insight_chain.run(
        metrics_str=str(metrics),
        top_products=products,
        ad_spend_roi=ads
    )
```

**4. 定时任务与推送**

```python
# scheduler.py
from apscheduler.schedulers.blocking import BlockingScheduler
from dingtalkchatbot.chatbot import DingtalkChatbot

webhook = "https://oapi.dingtalk.com/robot/send?access_token=XXX"
secret = "SECxxxx"
robot = DingtalkChatbot(webhook, secret=secret)

def job():
    report_md = build_report()  # 调用上述流程生成 Markdown
    robot.send_markdown(
        title="本周销售复盘报告",
        text=report_md,
        is_at_all=False
    )

scheduler = BlockingScheduler()
scheduler.add_job(job, "cron", day_of_week="mon", hour=9, minute=0)
scheduler.start()
```

---

## 效果

- **时间节省**：报告产出从 2 人天缩减到 **0.5 人天**（仍需人工确认异常），每周一 9 点准时推送
- **洞察质量**：运营负责人反馈 LLM 生成的 3-5 条结论中，约 **50-60% 可作为初稿参考**，剩余 30% 作为参考
- **异常响应**：当 GMV 周环比下跌超过 20% 时，Agent 会自动 @ 相关负责人并标红

---

## 关键要点

- **数据质量是上限**：建议先做 ETL 标准化，再接入 LLM；否则指标口径不一致会导致洞察失真
- **Prompt 结构化**：把指标、商品、广告三段数据分别填入模板，比让模型读原始表格更可控
- **人机结合**：复杂归因（如流量结构变化）仍需分析师二次确认，Agent 负责快速呈现全貌

---

## 延伸阅读

- [Pandas 官方文档](https://pandas.pydata.org/docs/)
- [LangChain 中文教程](https://www.langchain.com.cn/)
- [钉钉群机器人开发文档](https://open.dingtalk.com/document/isv/server-api/send-a-robot-group-message)

---

*Last updated: 2025-06-02*
