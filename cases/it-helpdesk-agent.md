---
title: 用 RAG + LLM 搭建企业内部 IT Helpdesk 自动排障 Agent
author: "@potato"
date: 2026-06
stack: LangChain, GPT-4o, ChromaDB, FastAPI, Slack API, Python
difficulty: 进阶
---

## 场景

200-500 人规模的企业中，IT 支持团队（通常 1-3 人）每天被大量重复问题淹没：

- "VPN 连不上了怎么办？"
- "邮箱客户端怎么配置？"
- "打印机显示离线"
- "GitLab 权限怎么申请？"
- "公司 Wi-Fi 密码是多少？"

这些问题 80% 有标准答案，但员工不知道去哪找文档，或者文档写了但看不懂。IT 人员每天重复回答同样的问题，真正有技术含量的工作（网络架构、安全策略）反而没时间做。

该 Agent 目标：**员工在 Slack/飞书提问后，Agent 自动检索内部知识库给出分步解决方案；常见问题可直接调用脚本自动修复**。

---

## 实现

### 整体架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Slack/飞书     │────▶│   FastAPI 服务   │────▶│  RAG 知识检索   │
│   员工提问      │     │  (Agent 主逻辑)  │     │  ChromaDB+Embedding│
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌─────────┐    ┌──────────┐    ┌──────────┐
        │ LLM 回答 │    │ 脚本执行 │    │ 工单创建  │
        │ (FAQ)   │    │ (自动修复)│    │ (人工兜底)│
        └─────────┘    └──────────┘    └──────────┘
```

### 核心配置

**1. 知识库构建（多源接入）**

```python
# knowledge_loader.py
from langchain.document_loaders import (
    NotionDirectoryLoader,
    ConfluenceLoader,
    TextLoader,
    UnstructuredMarkdownLoader
)
from langchain.text_splitter import MarkdownHeaderTextSplitter

SOURCES = {
    "notion": "data/notion_export/",
    "confluence": "https://wiki.company.com",
    "sop_files": "data/sop/*.md",
    "onboarding": "data/onboarding/"
}

def load_knowledge():
    docs = []
    # Notion 导出
    notion_loader = NotionDirectoryLoader(SOURCES["notion"])
    docs.extend(notion_loader.load())
    
    # Confluence（如有）
    # confluence_loader = ConfluenceLoader(url=...)
    
    # 本地 Markdown SOP
    for f in glob.glob(SOURCES["sop_files"]):
        loader = UnstructuredMarkdownLoader(f)
        docs.extend(loader.load())
    
    # 按 Markdown 标题拆分，保留结构
    splitter = MarkdownHeaderTextSplitter(headers_to_split_on=[("#", "Header 1"), ("##", "Header 2")])
    chunks = []
    for doc in docs:
        chunks.extend(splitter.split_text(doc.page_content))
    return chunks
```

**2. 向量化与检索**

```python
# vector_store.py
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    collection_name="it_knowledge",
    embedding_function=embeddings,
    persist_directory="./it_knowledge_db"
)

def index_documents(chunks):
    # 元数据中保留来源、最后更新时间
    metadatas = [{"source": c.metadata.get("source", "unknown"), 
                  "last_updated": c.metadata.get("last_updated", "2024-01-01")} 
                 for c in chunks]
    vectorstore.add_texts(
        texts=[c.page_content for c in chunks],
        metadatas=metadatas
    )
    vectorstore.persist()

def retrieve(query: str, k: int = 3):
    return vectorstore.similarity_search(query, k=k)
```

**3. Agent 主逻辑（判断 + 回答 + 脚本）**

```python
# agent.py
from langchain import OpenAI, PromptTemplate
from langchain.chains import LLMChain
import subprocess

# 定义可自动执行的脚本映射
AUTO_FIX_SCRIPTS = {
    "vpn_reset": "scripts/reset_vpn.sh",
    "printer_restart": "scripts/restart_print_spooler.ps1",
    "clear_dns": "scripts/flush_dns.sh",
}

IT_AGENT_PROMPT = PromptTemplate(
    input_variables=["context", "question"],
    template="""你是公司 IT 支持助手。根据以下内部文档，回答员工的技术问题。

已知信息：
{context}

员工问题：{question}

要求：
1. 用中文回答，语气友好
2. 给出分步操作指引（每步一行）
3. 如果涉及敏感操作（如删除数据、修改权限），必须说"请联系 IT 同事人工处理"
4. 如果已知信息不足以回答，说"这个问题我需要查一下，稍等"并建议创建工单

回答："""
)

llm = OpenAI(model_name="gpt-4o", temperature=0.1)
agent_chain = LLMChain(llm=llm, prompt=IT_AGENT_PROMPT)

def classify_intent(question: str) -> str:
    """判断是否需要自动执行脚本"""
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "system", "content": "判断以下 IT 问题是否可以通过执行预设脚本自动解决。输出：auto_fix / answer / escalate"},
                  {"role": "user", "content": question}]
    )
    return resp.choices[0].message.content.strip()

def handle_question(question: str, user_id: str) -> dict:
    intent = classify_intent(question)
    
    if intent == "auto_fix" and "VPN" in question.upper():
        # 执行脚本
        result = subprocess.run(["bash", AUTO_FIX_SCRIPTS["vpn_reset"]], capture_output=True, text=True)
        return {"type": "auto_fix", "message": f"已尝试自动修复 VPN，结果：{result.stdout}", "needs_follow_up": True}
    
    docs = retrieve(question)
    context = "\n---\n".join([d.page_content for d in docs])
    answer = agent_chain.run(context=context, question=question)
    
    # 如果置信度低，建议转人工
    if "我不知道" in answer or "查一下" in answer:
        create_ticket(user_id, question, context)
        return {"type": "escalate", "message": answer + "\n\n已为你创建 IT 工单，同事会尽快处理。"}
    
    return {"type": "answer", "message": answer}
```

**4. Slack 集成**

```python
# slack_bot.py
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

slack_app = App(token=os.environ["SLACK_BOT_TOKEN"])

@slack_app.event("app_mention")
def handle_mention(event, say):
    user_question = event["text"].replace(f"<@{slack_app.client.auth_test()['user_id']}>", "").strip()
    result = handle_question(user_question, event["user"])
    say(result["message"])

if __name__ == "__main__":
    handler = SocketModeHandler(slack_app, os.environ["SLACK_APP_TOKEN"])
    handler.start()
```

---

## 效果

在 300 人规模的科技公司中，预期效果：

| 指标 | 变化 |
|------|------|
| IT 重复问题占比 | 约 70% → 降至 30%（40% 由 Agent 直接解决） |
| 员工问题平均响应时间 | 30min（等 IT 同事看到） → 10s |
| IT 团队每日答疑时间 | 3h → 1h（处理复杂问题和工单） |
| 知识库文档使用率 | 几乎为零（没人翻） → 100%（每次回答都带引用来源） |
| 员工满意度 | IT 响应慢是常见投诉 → 即时响应满意度提升 |

---

## 要点

1. **知识库必须持续更新** — 软件版本升级后（如 macOS 新版本 VPN 配置变了），旧文档会误导用户。建议文档中标注"适用版本"，并建立过期提醒机制。

2. **自动执行脚本要谨慎** — 给用户自动重启打印服务是安全的，但自动改网络配置、删文件是危险的。建议所有脚本先经过安全审核，且高危操作必须人工确认。

3. **上下文保留很重要** — 如果用户说"还是不行"，Agent 需要知道他说的是上一个 VPN 问题。建议在 Slack thread 中维护对话上下文。

4. **不要完全替代人工** — Agent 解决不了的应该无缝转人工（创建工单或 @IT 同事），而不是让用户反复尝试无果后暴躁。

---

## 延伸阅读

- [LangChain RAG 教程](https://python.langchain.com/docs/use_cases/question_answering/)
- [Slack Bolt Python 框架](https://slack.dev/bolt-python/)
- [ChromaDB 持久化文档](https://docs.trychroma.com/usage-guide#persisting-data)

---

*Last updated: 2025-06-02*
