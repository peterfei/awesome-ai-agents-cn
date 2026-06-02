---
title: 用 LLM + RAG 搭建财报智能分析 Agent
author: "@potato"
date: 2026-06
stack: LangChain, Claude Sonnet, ChromaDB, Python, Streamlit
difficulty: 进阶
---

## 场景

投资机构/券商分析师每天需要阅读大量财报（A股、港股、美股），痛点在于：

1. **信息量大** — 一份年报 200-400 页，关键信息淹没在冗长表述中
2. **横向对比难** — 对比 5 家同行公司的毛利率变化，需要反复翻页
3. **非财务信息被忽略** — 管理层讨论、风险提示、ESG 报告中的信号
4. **时效性** — 季报季每天发布几百份报告，人手不够

目标是：**Agent 自动解析财报 PDF，提取关键指标，生成分析摘要，并支持自然语言问答**。

---

## 实现

### 架构

```
财报 PDF → 文本提取 → 结构化解析 → 向量存储 → RAG 问答
              ↓              ↓
         关键指标表     对比分析引擎
              ↓              ↓
         摘要生成       同业对比报告
```

### 核心实现

**第一步：PDF 解析与分块**

```python
import pymupdf  # fitz
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 解析 PDF
def extract_pdf_text(pdf_path):
    doc = pymupdf.open(pdf_path)
    text_blocks = []
    
    for page in doc:
        blocks = page.get_text("dict")["blocks"]
        for block in blocks:
            if "lines" in block:
                text = "".join([span["text"] for line in block["lines"] for span in line["spans"]])
                text_blocks.append({
                    "text": text,
                    "page": page.number,
                    "bbox": block["bbox"],
                    # 根据字体大小判断是否为标题
                    "is_heading": any(span["size"] > 14 for line in block["lines"] for span in line["spans"])
                })
    
    return text_blocks

# 语义分块：保留表格和段落完整性
def semantic_chunk(blocks):
    """将文本块按语义组织，表格单独保留"""
    chunks = []
    current_chunk = []
    
    for block in blocks:
        if "元" in block["text"] and "表" in block["text"][:20]:
            # 可能是表格标题，单独成块
            if current_chunk:
                chunks.append("\n".join(current_chunk))
                current_chunk = []
            chunks.append(block["text"])
        else:
            current_chunk.append(block["text"])
    
    return chunks

# 按章节打标签
def tag_sections(chunks):
    """识别章节类型，用于后续定向检索"""
    section_keywords = {
        "financial_summary": ["主要会计数据", "财务指标", "营业收入", "净利润"],
        "business_review": ["经营情况讨论", "管理层讨论", "业务回顾"],
        "risk_factors": ["风险提示", "风险因素", "可能面对的风险"],
        "corporate_governance": ["公司治理", "董监高", "股权激励"],
        "esg": ["环境", "社会责任", "ESG", "可持续发展"],
    }
    
    tagged = []
    for chunk in chunks:
        tags = []
        for section, keywords in section_keywords.items():
            if any(kw in chunk for kw in keywords):
                tags.append(section)
        tagged.append({"text": chunk, "tags": tags})
    
    return tagged
```

**第二步：关键指标结构化提取**

```python
from pydantic import BaseModel
from langchain.output_parsers import PydanticOutputParser

class FinancialMetrics(BaseModel):
    revenue: float  # 营业收入
    net_profit: float  # 净利润
    gross_margin: float  # 毛利率
    net_margin: float  # 净利率
    roe: float  # 净资产收益率
    debt_ratio: float  # 资产负债率
    operating_cash_flow: float  # 经营现金流
    yoy_revenue_growth: float  # 营收同比增长
    yoy_profit_growth: float  # 净利同比增长

parser = PydanticOutputParser(pydantic_object=FinancialMetrics)

# 用 LLM 从文本中提取指标
extract_prompt = """
从以下财报文本中提取关键财务指标。如果某指标未提及，留空。

文本：
{text}

{format_instructions}
"""

# 批量提取所有指标
metrics_list = []
for report in reports:
    text = extract_key_sections(report)
    result = llm.predict(extract_prompt.format(
        text=text,
        format_instructions=parser.get_format_instructions()
    ))
    metrics_list.append(parser.parse(result))
```

**第三步：向量存储与 RAG**

```python
from langchain_chroma import Chroma
from langchain.embeddings import OpenAIEmbeddings

# 初始化向量库
vectorstore = Chroma(
    collection_name="financial_reports",
    embedding_function=OpenAIEmbeddings(),
    persist_directory="./chroma_db"
)

# 存入带标签的文档
for tagged in tagged_chunks:
    vectorstore.add_texts(
        texts=[tagged["text"]],
        metadatas=[{
            "company": company_name,
            "report_date": report_date,
            "tags": tagged["tags"],
            "page": tagged.get("page", 0)
        }]
    )

# RAG 问答链
from langchain.chains import RetrievalQA

qa_chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4o"),
    retriever=vectorstore.as_retriever(
        search_kwargs={
            "k": 5,
            "filter": {"company": target_company}  # 可筛选特定公司
        }
    ),
    return_source_documents=True
)

# 使用示例
response = qa_chain.invoke(""
""
请分析这家公司近两年的毛利率变化趋势，并解释变化原因。
请引用原文出处（页码）。
""")
```

**第四步：同业对比与报告生成**

```python
def generate_comparison_report(companies, metrics_list):
    """生成横向对比分析报告"""
    
    df = pd.DataFrame([m.dict() for m in metrics_list])
    df["company"] = companies
    
    comparison_prompt = f"""
    你是一位资深行业分析师。以下是 {len(companies)} 家公司的最新财报核心指标：

    {df.to_markdown()}

    请生成一份对比分析摘要，包含：
    1. 各公司盈利能力排序及点评
    2. 成长性（营收/净利增速）排名
    3. 财务健康度（现金流、负债）评估
    4. 每家公司最值得关注的亮点和风险点
    5. 对投资者的综合建议

    要求：专业、客观、数据驱动，每句话尽量有数据支撑。
    """
    
    return llm.predict(comparison_prompt)
```

**第五步：Streamlit 交互界面**

```python
import streamlit as st

st.title("财报智能分析助手")

# 上传 PDF
uploaded = st.file_uploader("上传财报 PDF", type="pdf", accept_multiple_files=True)

# 选择分析维度
analysis_type = st.selectbox("分析类型", [
    "单公司深度分析",
    "同业对比分析",
    "自定义问答"
])

# 问答模式
if analysis_type == "自定义问答":
    question = st.text_input("输入你的问题", "这家公司的核心竞争力是什么？")
    if question:
        with st.spinner("分析中..."):
            result = qa_chain.invoke(question)
            st.write(result["result"])
            
            # 展示引用来源
            st.subheader("引用来源")
            for doc in result["source_documents"]:
                with st.expander(f"第 {doc.metadata['page']} 页"):
                    st.write(doc.page_content[:500])
```

---

## 效果

在管理 20 亿规模量级的投资研究场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 单份财报阅读时间 | 4h → 30min（AI 预读 + 人重点review） |
| 覆盖公司数量 | 20 家 → 80 家 |
| 关键指标提取准确率 | 92%（需人工复核） |
| 同业对比报告产出 | 2 天 → 2 小时 |
| 研究员满意度 | 85%（认为 AI 摘要减少了重复劳动） |

**典型场景**：研究员问"这家公司近两年的应收账款周转天数变化及原因"，Agent 自动检索财报中"应收账款"相关段落，结合管理层讨论生成带引用页码的分析。

---

## 要点

1. **PDF 解析是瓶颈** — 财报 PDF 格式各异（扫描件、复杂表格、图文混排），直接提取文字会错位。建议用 `pymupdf` + `unstructured` 组合，对表格单独处理（转 HTML 或 CSV）。

2. **指标标准化** — 不同公司财报的科目命名不同（"营业收入"vs"主营业务收入"），需要建立别名映射表。LLM 提取后必须人工 spot check，尤其是"亿"和"万"的单位容易搞混。

3. **引用溯源必须准确** — 投资研究对信息来源要求极高。RAG 返回的结果必须标注"第 X 页"，方便研究员复核。建议用原文 chunk 作为上下文，不要让 LLM 自由发挥。

4. **时效性处理** — 财报有"业绩预告→正式报告→更正公告"的序列，向量库需要版本管理，确保问答时引用的是最新版。

---

## 扩展思路

- 接入实时股价数据，做"财报发布后市场反应"的联动分析
- 情感分析：分析管理层讨论中的措辞变化（从"充满信心"到"面临挑战"）
- 产业链上下游联动：自动提取供应商/客户信息，构建产业图谱
- 监管问询关联：把证监会问询函和回复也纳入 RAG，分析公司合规风险
- 多语言支持：港股中英文对照、美股英文财报自动翻译分析

---

## 相关资源

- [pymupdf — PDF 解析](https://pymupdf.readthedocs.io)
- [LangChain RAG 教程](https://python.langchain.com/docs/use_cases/question_answering/)
- [ChromaDB 向量数据库](https://www.trychroma.com)
- [中国证监会信息披露平台](http://www.cninfo.com.cn)
- [Streamlit 官方文档](https://docs.streamlit.io)
