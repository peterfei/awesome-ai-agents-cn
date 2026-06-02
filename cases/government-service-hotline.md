---
title: 用 RAG + 大模型搭建政务服务热线智能答复 Agent
author: "@potato"
date: 2026-06
stack: Dify, DeepSeek, RAG, PostgreSQL, 政务中台 API
difficulty: 进阶
---

## 场景

12345 政务服务热线和区县政务大厅每天接到海量咨询：

1. **高频重复问题占比高** — "怎么办居住证？""社保断缴怎么补？"占 60%+
2. **政策更新快** — 新政策出台后接线员培训跟不上
3. **跨部门知识壁垒** — 人社、税务、公安的政策接线员不一定懂
4. **服务时间有限** — 夜间、节假日只能留言，无法即时答复

目标是：**Agent 7x24 小时自动答复常见政策咨询，复杂问题精准转接对应部门，并辅助接线员实时查询政策**。

---

## 实现

### 架构

```
市民咨询 → 语音/文字输入 → 意图识别 → ① 高频政策 → RAG 检索 → 自动答复
                                    → ② 复杂个案 → 转接人工 + 推送相关政策摘要
                                    → ③ 投诉建议 → 自动分类立案 → 工单系统
                                    → ④ 紧急事件 → 触发应急流程（110/120联动）
```

### 核心配置

**第一步：政务知识库建设**

```python
# 知识库来源
data_sources = [
    "各局委办政策文件（PDF/Word）",
    "办事指南（一网通办页面）",
    "历史工单高频问题及答复",
    "法律法规原文",
    "街道/社区本地化办事流程"
]

# 文档处理与向量化
from langchain.document_loaders import DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_chroma import Chroma

loader = DirectoryLoader("/data/policy_docs", glob="**/*.pdf")
docs = loader.load()

# 政务文档特殊分块策略：按"条款"粒度切分
splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=100,
    separators=["\n第[一二三四五六七八九十百]+章", "\n第[一二三四五六七八九十百]+条", "\n", "。", ""]
)

chunks = splitter.split_documents(docs)

# 存入向量库 + 元数据（部门、生效日期、废止状态）
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=OpenAIEmbeddings(),
    metadatas=[{
        "dept": extract_department(doc),
        "effective_date": extract_date(doc),
        "is_active": not is_obsolete(doc),
        "doc_type": "政策文件"
    } for doc in chunks]
)
```

**第二步：RAG 检索与答复生成**

```python
def answer_citizen_query(query, citizen_info=None):
    """市民咨询主流程"""
    
    # 1. 意图识别
    intent = classify_intent(query)
    
    if intent == "emergency":
        return {"type": "emergency", "action": "transfer_to_110"}
    
    # 2. 多路检索
    # 基础 RAG
    base_docs = vectorstore.similarity_search(query, k=5, 
        filter={"is_active": True})
    
    # 历史工单相似案例
    case_docs = case_store.similarity_search(query, k=3)
    
    # 3. 生成答复
    context = format_docs(base_docs + case_docs)
    
    prompt = f"""
    你是政务服务中心的智能助手。请根据以下政策知识，回答市民问题。

    ## 检索到的政策依据
    {context}

    ## 市民问题
    {query}

    ## 要求
    1. 先给出直接答案（能办/不能办/需补充材料）
    2. 列出所需材料清单（如有）
    3. 说明办理地点/线上入口
    4. 给出办理时限承诺
    5. 引用政策文件名及条款（如"根据《居住证暂行条例》第 X 条"）
    6. 如果政策有地区差异，标注"具体以 XX 区政务大厅为准"
    7. 如果知识库中没有明确答案，诚实说明"该问题需人工核实"
    8. 绝不编造政策内容
    """
    
    response = llm.predict(prompt)
    
    # 4. 判断是否需要人工
    needs_human = check_if_needs_human(response, query)
    
    return {
        "answer": response,
        "sources": [d.metadata for d in base_docs],
        "needs_human": needs_human,
        "suggested_dept": base_docs[0].metadata["dept"] if base_docs else None
    }
```

**第三步：工单自动分类与流转**

```python
def auto_classify_ticket(query, answer_result):
    """自动生成工单并路由到对应部门"""
    
    prompt = f"""
    请对以下市民诉求进行分类：
    
    市民原话：{query}
    初步答复：{answer_result['answer']}
    
    输出 JSON：
    {{
      "category": "咨询/投诉/求助/建议/举报",
      "dept": "主办部门（人社局/税务局/公安局/街道办等）",
      "urgency": "普通/加急/特急",
      "keywords": ["关键词1", "关键词2"],
      "summary": "一句话摘要",
      "is_repeat": true/false  // 是否重复投诉
    }}
    """
    
    classification = json.loads(llm.predict(prompt))
    
    # 创建工单
    ticket = create_ticket(
        category=classification["category"],
        dept=classification["dept"],
        urgency=classification["urgency"],
        summary=classification["summary"],
        citizen_query=query,
        ai_answer=answer_result["answer"],
        status="待处理" if classification["urgency"] != "特急" else "已告警"
    )
    
    # 特急自动通知值班负责人
    if classification["urgency"] == "特急":
        notify_duty_officer(ticket)
    
    return ticket
```

**第四步：接线员辅助（Copilot 模式）**

```python
class AgentCopilot:
    """实时辅助人工接线员"""
    
    def on_call(self, transcript):
        """通话过程中实时提示"""
        
        # 实时检索相关政策
        docs = vectorstore.similarity_search(transcript, k=3)
        
        # 判断市民情绪
        sentiment = analyze_sentiment(transcript)
        
        # 生成辅助提示
        suggestions = []
        if sentiment == "negative":
            suggestions.append("⚠️ 市民情绪较激动，建议先安抚")
        
        suggestions.append(f"📋 相关政策：{docs[0].page_content[:200]}...")
        suggestions.append(f"🏢 建议转接部门：{docs[0].metadata['dept']}")
        
        # 如果检测到高频关键词，提示"该问题有标准答复"
        if is_frequent_question(transcript):
            suggestions.append("💡 该问题标准答复：" + get_standard_answer(transcript))
        
        return suggestions
```

---

## 效果

在区级政务服务中心场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 7x24 自动答复率 | 62%（夜间+节假日主要由 Agent 接待） |
| 接线员平均通话时长 | 6min → 4min（AI 辅助实时提示） |
| 首次响应时间 | 平均 30s → 平均 5s（自动） |
| 工单分类准确率 | 88% |
| 市民满意度 | 78% → 86% |
| 接线员培训周期 | 2 个月 → 2 周（AI 辅助降低记忆负担） |

---

## 要点

1. **政策知识库必须及时更新** — 政策文件一有变动（如新文件替代旧文件），向量库需要立即更新，否则 Agent 会引用废止文件给出错误答复。建议建立"文件生效/废止"自动同步机制。

2. **地域差异要处理好** — "怎么办居住证"在 A 区和 B 区的材料清单可能不同。检索时需要结合市民归属地过滤，或明确标注"请以您所在街道要求为准"。

3. **答复必须有政策出处** — 政务答复的严肃性要求每句话都有依据。Agent 输出必须包含引用的文件名和条款号，不能凭空下结论。

4. **敏感词拦截** — 涉及群体性事件、信访、涉稳信息的咨询，必须触发人工接管流程，不能自动答复。

---

## 扩展思路

- 接入"一网通办"API，实现"问答即办理"（Agent 答复后直接跳转线上填报）
- 多语言支持：服务外籍人士，自动切换英语/日语/韩语答复
- 智能回访：工单办结后 AI 自动电话回访满意度
- 舆情监测：自动聚类高频投诉主题，生成"民生热点周报"供领导决策

---

## 相关资源

- [Dify 知识库 RAG 配置](https://docs.dify.ai/guides/knowledge-base)
- [DeepSeek 官方 API](https://platform.deepseek.com)
- [中国政府网政策文件库](https://www.gov.cn/zhengce/index.htm)
- [全国一体化政务服务平台](https://www.gjzwfw.gov.cn)
