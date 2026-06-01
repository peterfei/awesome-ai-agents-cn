---
title: 用 LangGraph 搭建 Text-to-SQL 查询 Agent
author: @peterfei
date: 2026-05
stack: LangGraph, Claude Sonnet, PostgreSQL, LangSmith
difficulty: 进阶
---

## 场景

运营团队每天需要查数据，但不会写 SQL。每次提需求给开发："帮我查一下上周的订单数据" → 开发排期 → 两天后给结果。

这个流程的问题：
1. 开发时间被碎片化查询打断
2. 运营等不及，自己猜数据做决策
3. 同样的问题重复问

目标是：**让运营用自然语言直接查数据库，Agent 负责把中文翻译成 SQL 并返回结果**。

---

## 实现

### 架构

```
用户输入 → 意图识别 → Schema 匹配 → SQL 生成 → SQL 执行 → 结果解释
                              ↑                    ↓
                        错误修正 ←———— 执行失败
```

### 核心实现

使用 LangGraph 构建有状态的多步骤 Agent：

```python
from langgraph.graph import StateGraph
from typing import TypedDict, Optional

class AgentState(TypedDict):
    question: str           # 用户原始问题
    schema_context: str     # 相关表结构
    sql: Optional[str]      # 生成的 SQL
    result: Optional[str]   # 执行结果
    error: Optional[str]    # 错误信息
    retry_count: int        # 重试次数

# 定义 Agent 节点
def extract_intent(state: AgentState):
    """第一步：识别用户意图，确定需要查询哪些表"""
    # 用 LLM 分析用户问题，匹配数据库表名和字段
    # 输出：相关的表结构上下文
    pass

def generate_sql(state: AgentState):
    """第二步：根据表结构生成 SQL"""
    prompt = f"""
    根据以下 PostgreSQL 表结构，将中文问题转换为 SQL：

    表结构：
    {state['schema_context']}

    问题：{state['question']}

    要求：
    - 只做 SELECT 查询
    - 使用中文别名（方便运营看懂）
    - 如果问题涉及复杂统计，用窗口函数
    - 不要做 UPDATE/DELETE
    """
    # 调用 LLM 生成 SQL
    pass

def execute_sql(state: AgentState):
    """第三步：执行 SQL"""
    try:
        result = db.execute(state['sql'])
        return {"result": result}
    except Exception as e:
        return {"error": str(e)}

def fix_sql(state: AgentState):
    """第四步：如果执行失败，根据错误信息修正 SQL"""
    if state['retry_count'] >= 3:
        return {"error": "重试次数超过上限"}

    # 用 LLM 分析错误并修正 SQL
    pass

def explain_result(state: AgentState):
    """第五步：把查询结果转换成自然语言"""
    prompt = f"""
    用户问：{state['question']}
    SQL：{state['sql']}
    查询结果：{state['result']}

    请用中文简洁回答用户的问题，附上关键数据。
    """
    pass

# 构建图
graph = StateGraph(AgentState)
graph.add_node("extract_intent", extract_intent)
graph.add_node("generate_sql", generate_sql)
graph.add_node("execute_sql", execute_sql)
graph.add_node("fix_sql", fix_sql)
graph.add_node("explain_result", explain_result)

graph.add_edge("extract_intent", "generate_sql")
graph.add_edge("generate_sql", "execute_sql")
graph.add_conditional_edges(
    "execute_sql",
    lambda state: "fix_sql" if state.get("error") else "explain_result"
)
graph.add_edge("fix_sql", "execute_sql")
```

### 部署

使用 LangGraph 的 `serve` 模式部署为 API：

```bash
langgraph serve
# → http://localhost:8000

# 调用示例
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "上个月销量前十的商品是什么？"}'

# 响应
{
  "question": "上个月销量前十的商品是什么？",
  "sql": "SELECT p.name AS 商品名, COUNT(o.id) AS 销量 FROM orders o JOIN products p ON o.product_id = p.id WHERE o.created_at >= date_trunc('month', CURRENT_DATE - INTERVAL '1 month') AND o.created_at < date_trunc('month', CURRENT_DATE) GROUP BY p.name ORDER BY 销量 DESC LIMIT 10",
  "result": "1. 无线耳机 2345单\n2. 手机壳 1892单\n...",
  "answer": "上个月销量前十的商品中，无线耳机以 2345 单排名第一，其次是手机壳（1892 单）。"
}
```

---

## 效果

在一个月交易量 10 万+ 的电商数据库上跑了两周：

| 指标 | 数据 |
|------|------|
| SQL 生成准确率 | 82%（首轮生成） |
| 自修正后准确率 | 91%（最多 2 次重试后） |
| 平均查询耗时 | 8s（含 LLM 调用） |
| 运营自主查询占比 | 从 0% → 43% |
| 开发被查询打断次数 | 减少 60% |

**失败的典型场景**：
- 涉及多表 JOIN + 复杂 WHERE 条件时容易出错
- "环比增长"这类需要自查询嵌套的理解不够好
- 表名和字段名不规范的数据库（`col1`、`fld2` 这种）效果差

---

## 要点

1. **只读保护** — 这个 Agent 绝对不能有写权限。连接数据库的用户需要是只读角色，并且在 SQL 执行前再做一次正则检查（过滤 DELETE/UPDATE/DROP/ALTER）。

2. **Schema 上下文裁剪** — 大型数据库可能有几百张表，不能把所有表结构都塞进 prompt。需要根据用户问题先匹配相关表（通过表名相似度 + 外键关系），只把相关表的 schema 发给 LLM。

3. **错误修正是核心** — 一次生成正确的 SQL 很难（82% 准确率），但有了执行-报错-修正这个循环后，准确率可以显著提升。每一轮修正都在 LangSmith 中记录，用于后续分析。

4. **结果解释比 SQL 重要** — 运营不在乎 SQL 写得多漂亮，他们在乎答案。把结果翻译成自然语言 + 可视化的表格，比只返回一个 JSON 数组有用得多。

---

## 扩展思路

- 加入缓存策略：相同问题 24 小时内命中缓存（很多报表问题是重复的）
- 可视化集成：自动判断数据适合折线图还是柱状图，返回图表
- 敏感数据脱敏：电话号码、邮箱自动打码

---

## 相关资源

- [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)
- [LangSmith 可观测性](https://smith.langchain.com)
- [Text-to-SQL 论文综述](https://arxiv.org/abs/2305.09306)
