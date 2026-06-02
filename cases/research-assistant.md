# 用 LangChain + RAG 搭建学术文献综述与研究助手 Agent

> 标签：`知识管理` `RAG` `LangChain` `学术` `Zotero` `OpenAI`

---

## 场景

某高校科研团队与某药企研发部门均有类似痛点：

- **文献爆炸**：每周新增几十篇相关领域论文，人工筛选耗时
- **知识碎片化**：读过的论文散落在 Zotero、Notion、本地 PDF 中，想复用时找不到
- **综述写作痛苦**：写开题报告或项目申请书时，需要把几十篇文献的观点串成逻辑线，往往花 1-2 周

该 Agent 目标：**自动追踪指定主题的 arXiv / PubMed 新论文，一键生成带引用的综述草稿，并支持自然语言问答**。

---

## 实现

### 整体架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   文献采集层    │────▶│   向量化知识库   │────▶│   综述生成引擎   │
│  arXiv/PubMed   │     │  ChromaDB+Qwen  │     │  LangChain+GPT  │
│   Zotero API    │     │   (RAG Pipeline)│     │  (Map-Reduce)   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                                               │
         └─────────────── 问答接口 ───────────────────────┘
                        (Streamlit)
```

### 核心配置

**1. 文献采集与解析**

```python
# paper_fetcher.py
import arxiv
from pymed import PubMed
from langchain.document_loaders import PyPDFLoader

def fetch_arxiv(query: str, max_results: int = 10) -> list:
    client = arxiv.Client()
    search = arxiv.Search(
        query=query,
        max_results=max_results,
        sort_by=arxiv.SortCriterion.SubmittedDate
    )
    papers = []
    for result in client.results(search):
        papers.append({
            "title": result.title,
            "abstract": result.summary,
            "pdf_url": result.pdf_url,
            "published": result.published,
            "authors": [a.name for a in result.authors]
        })
    return papers
```

**2. 向量化入库（含元数据）**

```python
# vector_store.py
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.text_splitter import RecursiveCharacterTextSplitter

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    collection_name="research_papers",
    embedding_function=embeddings,
    persist_directory="./chroma_db"
)

def index_papers(papers: list):
    splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
    for p in papers:
        # 摘要 + 标题作为文本
        text = f"Title: {p['title']}\nAbstract: {p['abstract']}"
        chunks = splitter.split_text(text)
        vectorstore.add_texts(
            texts=chunks,
            metadatas=[{
                "title": p["title"],
                "authors": ", ".join(p["authors"]),
                "published": str(p["published"]),
                "pdf_url": p["pdf_url"],
                "source": "arxiv"
            }] * len(chunks)
        )
```

**3. 综述生成（Map-Reduce 模式）**

```python
# review_generator.py
from langchain.chains.summarize import load_summarize_chain
from langchain import OpenAI, PromptTemplate

MAP_PROMPT = PromptTemplate(
    input_variables=["text"],
    template="""请用中文总结以下论文摘要的核心贡献（1-2 句话），并指出其研究方法：

{text}

总结："""
)

REDUCE_PROMPT = PromptTemplate(
    input_variables=["text"],
    template="""你正在撰写一份文献综述。请将以下各篇论文的总结整合为一篇逻辑连贯的综述段落，按主题归类，并指出研究趋势与空白：

{text}

综述："""
)

llm = OpenAI(temperature=0.3, model_name="gpt-4o")
chain = load_summarize_chain(
    llm, chain_type="map_reduce",
    map_prompt=MAP_PROMPT,
    combine_prompt=REDUCE_PROMPT,
    verbose=True
)

def generate_review(papers: list) -> str:
    from langchain.schema import Document
    docs = [Document(page_content=p["abstract"], metadata=p) for p in papers]
    return chain.run(docs)
```

**4. 问答接口（带引用溯源）**

```python
# qa_app.py
import streamlit as st
from langchain.chains import RetrievalQA

st.set_page_config(page_title="研究助手", layout="wide")

qa_chain = RetrievalQA.from_chain_type(
    llm=OpenAI(model_name="gpt-4o", temperature=0.2),
    chain_type="stuff",
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    return_source_documents=True
)

query = st.text_input("输入研究问题")
if query:
    result = qa_chain({"query": query})
    st.markdown(result["result"])
    st.subheader("参考来源")
    for doc in result["source_documents"]:
        st.markdown(f"- *{doc.metadata['title']}* ({doc.metadata['authors']})")
```

---

## 效果

- **筛选效率**：研究团队每周读论文时间可从 **8 小时** 降至 **2-3 小时**（只看 Agent 推荐的高相关度论文）
- **综述产出**：输入 20 篇论文，Agent 可在 **3 分钟** 内生成一篇 800 字的中文综述草稿，准确率约 **60-75%**（需人工复核）
- **知识复用**：通过自然语言问答，过去读过的论文可以被"激活"，团队反馈"像多了一个科研助理"

---

## 关键要点

- **元数据保留**：入库时必须保留标题、作者、发表日期、PDF 链接，否则生成的综述无法引用
- **主题限定**：检索式要写清楚（如 `LLM AND drug discovery`），否则 arXiv 的噪音很大
- **Map-Reduce 优于单轮**：单轮直接喂 20 篇摘要给 GPT 会超出上下文或丢失细节，Map-Reduce 更稳定

---

## 延伸阅读

- [LangChain Map-Reduce 文档](https://python.langchain.com/docs/modules/chains/document/combine_docs#mapreduce)
- [arXiv API 使用指南](https://info.arxiv.org/help/api/)
- [Zotero API 文档](https://www.zotero.org/support/dev/web_api/v3/start)

---

*Last updated: 2025-06-02*
