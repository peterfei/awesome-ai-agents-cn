---
title: 用 LangGraph 搭建多 Agent 协作内容生产流水线
author: "@potato"
date: 2026-06
stack: LangGraph, Claude Sonnet, Tavily, DALL-E, Airtable
difficulty: 专家
---

## 场景

内容团队（公众号/短视频/电商详情页）的生产流程涉及多个角色：

1. **选题** — 编辑找热点
2. **调研** — 查资料、找数据
3. **撰稿** — 写正文
4. **配图** — 找图/做图
5. **审核** — 校对、合规检查
6. **排版** — 适配各平台

传统模式是串行人工处理，一篇内容 1-2 天。Agent 可以做什么？

目标是：**多个 Agent 协作，选题→调研→撰稿→配图→审核→排版全自动流水线，人工只做终审**。

---

## 实现

### 架构

```
触发器 → 选题 Agent → 调研 Agent → 撰稿 Agent → 配图 Agent → 审核 Agent → 排版 Agent
            ↓            ↓            ↓            ↓            ↓            ↓
         热点扫描      资料收集      正文生成      图片生成      质量检查      多平台适配
```

### 核心实现

**LangGraph 多 Agent 状态定义**

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List, Optional

class ContentState(TypedDict):
    topic: Optional[str]           # 选题
    research_notes: Optional[str]  # 调研笔记
    draft: Optional[str]          # 初稿
    images: Optional[List[str]]   # 配图链接
    review_result: Optional[dict]  # 审核结果
    final_outputs: Optional[dict]    # 各平台终稿
    status: str                   # 当前状态

# 初始化
graph = StateGraph(ContentState)
```

**Agent 1：选题 Agent**

```python
def topic_agent(state: ContentState):
    """扫描热点，生成选题建议"""
    
    # 多渠道热点采集
    sources = {
        "weibo_hot": fetch_weibo_hot(),
        "zhihu_hot": fetch_zhihu_hot(),
        "github_trending": fetch_github_trending(),
        "industry_news": fetch_rss_feeds()
    }
    
    prompt = f"""
    你是资深内容主编。以下是今日热点数据：
    {json.dumps(sources, ensure_ascii=False)}

    请根据你们账号的定位（AI/科技/开发者），选出 TOP3 选题，并说明理由。
    每个选题需包含：
    - 标题方向（吸引人但不标题党）
    - 目标读者
    - 独特角度（为什么值得写）
    - 预期数据来源
    """
    
    result = call_llm(prompt)
    return {"topic": result, "status": "researched"}
```

**Agent 2：调研 Agent**

```python
def research_agent(state: ContentState):
    """深度调研，收集事实和数据"""
    
    topic = state["topic"]
    
    # Tavily 搜索
    search_results = tavily.search(
        query=f"{topic} 最新数据 2025 2026",
        max_results=10
    )
    
    # 提取关键信息
    prompt = f"""
    你是研究助理。请基于以下搜索结果，整理一份调研笔记。

    选题：{topic}
    搜索结果：
    {format_search_results(search_results)}

    要求：
    1. 提取 3-5 个核心事实（带数据来源）
    2. 找到 1-2 个有代表性的案例/数据
    3. 标注信息的可信度（官方数据/媒体报道/自媒体）
    4. 如果发现矛盾信息，标注争议点
    5. 整理成 Markdown 笔记格式
    """
    
    notes = call_llm(prompt)
    return {"research_notes": notes, "status": "drafting"}
```

**Agent 3：撰稿 Agent**

```python
def writer_agent(state: ContentState):
    """基于调研笔记生成正文"""
    
    prompt = f"""
    你是技术作者。请根据以下调研笔记撰写文章。

    选题：{state['topic']}
    调研笔记：
    {state['research_notes']}

    写作要求：
    1. 字数 1500-2000 字
    2. 开头用钩子（数据/冲突/问题）
    3. 正文分 3-4 个小标题
    4. 每个论点有数据支撑
    5. 结尾总结 + 引导互动
    6. 风格：专业但通俗，像和朋友聊天
    7. 不要编造数据，没数据的观点标[待核实]
    """
    
    draft = call_llm(prompt)
    return {"draft": draft, "status": "illustrating"}
```

**Agent 4：配图 Agent**

```python
def illustrator_agent(state: ContentState):
    """生成文章配图"""
    
    # 从文章中提取适合配图的场景
    prompt = f"""
    以下是一篇文章。请判断需要几张配图，并为每张图写 DALL-E prompt。

    文章：
    {state['draft']}

    要求：
    1. 封面图：1 张，概念性、吸引眼球
    2. 正文插图：每 500 字配 1 张示意图或数据图
    3. DALL-E prompt 要包含风格描述（扁平插画/写实/科技感）
    4. 避免生成文字（AI 画图文字易错），图中不含中文
    """
    
    image_plan = call_llm(prompt)
    
    # 调用 DALL-E 生成图片
    images = []
    for item in image_plan["images"]:
        if item["type"] == "chart" and item.get("data"):
            # 数据图用代码生成（matplotlib）
            img_url = generate_chart(item["data"])
        else:
            # 概念图用 DALL-E
            img_url = dalle.generate(item["prompt"], size="1024x1024")
        images.append(img_url)
    
    return {"images": images, "status": "reviewing"}
```

**Agent 5：审核 Agent**

```python
def reviewer_agent(state: ContentState):
    """质量审核：事实核查 + 合规检查 + 风格检查"""
    
    prompt = f"""
    你是内容主编，请对以下文章进行终审。

    文章：
    {state['draft']}

    审核维度：
    1. 事实核查：数据是否有来源？是否存在可疑数字？
    2. 合规检查：有无违禁词/极限词？是否涉及敏感话题？
    3. 风格检查：是否太AI味？是否有错别字？
    4. 结构检查：开头是否吸引人？结尾是否有行动引导？
    5. 配图检查：图文是否匹配？图片有无 inappropriate 内容？

    输出 JSON：
    {{
      "pass": true/false,
      "score": 1-10,
      "issues": [{{"type": "事实/合规/风格", "desc": "", "severity": "high/medium/low"}}],
      "revision_suggestions": "修改建议"
    }}
    """
    
    review = json.loads(call_llm(prompt))
    return {"review_result": review, "status": "formatting" if review["pass"] else "revising"}
```

**Agent 6：排版 Agent**

```python
def formatter_agent(state: ContentState):
    """多平台格式适配"""
    
    draft = state["draft"]
    
    formats = {
        "wechat": format_for_wechat(draft, state["images"]),
        "zhihu": format_for_zhihu(draft),
        "juejin": format_for_juejin(draft),
        "twitter": generate_thread(draft)
    }
    
    return {"final_outputs": formats, "status": "done"}

def format_for_wechat(text, images):
    """公众号格式：特定标题样式、引导关注、卡片"""
    return f"""
    <h1 style="...">{extract_title(text)}</h1>
    {insert_images(text, images)}
    <p style="...">觉得有用？点个「在看」👇</p>
    """
```

**LangGraph 状态流转**

```python
graph.add_node("topic", topic_agent)
graph.add_node("research", research_agent)
graph.add_node("writer", writer_agent)
graph.add_node("illustrator", illustrator_agent)
graph.add_node("reviewer", reviewer_agent)
graph.add_node("formatter", formatter_agent)

graph.add_edge("topic", "research")
graph.add_edge("research", "writer")
graph.add_edge("writer", "illustrator")
graph.add_edge("illustrator", "reviewer")

# 审核不通过则返回修改
graph.add_conditional_edges(
    "reviewer",
    lambda state: "writer" if not state["review_result"]["pass"] else "formatter"
)

graph.add_edge("formatter", END)

# 编译并运行
app = graph.compile()
result = app.invoke({"status": "init"})
```

---

## 效果

在科技媒体内容团队场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 单篇内容生产周期 | 2 天 → 4 小时（人工确认后发布） |
| 日产量 | 1 篇 → 3 篇 |
| 审核不返工率 | 65%（初稿即达标） |
| 编辑工作时间分配 | 70% 写作 → 30% 写作 + 50% 选题策划 + 20% 社区互动 |
| 多平台分发覆盖率 | 人工发 2 个平台 → AI 自动适配 5 个平台 |

**翻车案例**：
- 调研 Agent 引用了一篇自媒体文章的数据，但审核 Agent 没发现来源可疑，发布后被人指出数据错误
- 修复：调研 Agent 增加规则"优先引用官方/权威来源，自媒体数据标注[待核实]"

---

## 要点

1. **Agent 间通信要结构化** — 不要让 Agent 之间传自由文本，用 JSON/状态字段保证信息不丢失。调研 Agent 的输出必须包含"来源列表"，撰稿 Agent 必须引用来源。

2. **审核 Agent 是守门员** — 宁可严格也不放过。审核不通过时，要明确标注哪个 Agent 负责修改（是事实问题→调研重写，还是风格问题→撰稿修改）。

3. **人工终审不能省** — 全自动发布风险高。建议流水线跑到"排版完成"后暂停，人工一键确认再发布。给编辑"接受/拒绝/修改"三个按钮。

4. **反馈闭环** — 发布后把真实数据（阅读量、互动率）回流，用来评估哪些 Agent 产出效果好。比如配图 Agent 生成的封面图，用 A/B 测试数据来优化 prompt。

---

## 扩展思路

- 接入实时热点 API， Agent 自动判断"现在是不是发这篇的好时机"
- 读者画像联动：根据粉丝画像调整选题方向和写作风格
- 竞品监控 Agent：监控竞品账号更新，自动对比内容差异
- SEO 优化 Agent：在撰稿阶段自动插入关键词、优化标题
- 评论互动 Agent：发布后自动回复读者评论，维持互动

---

## 相关资源

- [LangGraph 多 Agent 教程](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/)
- [Tavily AI 搜索 API](https://tavily.com)
- [DALL-E 图像生成 API](https://platform.openai.com/docs/guides/images)
- [n8n 自动化工作流](https://docs.n8n.io)
