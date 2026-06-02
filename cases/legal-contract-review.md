---
title: 用 RAG + LLM 搭建合同智能审查 Agent
author: "@potato"
date: 2026-06
stack: LangChain, Claude Sonnet, ChromaDB, Python, Streamlit
difficulty: 专家
---

## 场景

企业法务/律师每天审大量合同，痛点：

1. **重复劳动** — 采购合同、NDA、劳动合同等 80% 是标准条款，但仍需逐条核对
2. **遗漏风险** — 高强度工作下容易漏掉关键风险点（违约金过低、管辖条款不利）
3. **知识沉淀难** — 资深律师的经验没法系统化传承给 junior
4. **对比耗时** — 对比合同稿和内部模板差异，纯人肉 diff

目标是：**Agent 自动对比合同与模板、标记风险条款、给出修改建议，法务从"逐条看"变为"重点审"**。

---

## 实现

### 架构

```
合同 PDF/Word → 文本提取 → 条款结构化 → RAG 匹配模板
                                    ↓
                             ① 缺失条款标记
                             ② 风险条款识别
                             ③ 修改建议生成
                                    ↓
                             法务审核工作台
```

### 核心实现

**第一步：合同条款结构化提取**

```python
from langchain.output_parsers import PydanticOutputParser
from pydantic import BaseModel

class ContractClause(BaseModel):
    title: str  # 条款标题
    type: str   # 条款类型（付款/交付/违约/保密/争议解决...）
    content: str
    page: int
    risk_level: str = "pending"  # 待评估

parser = PydanticOutputParser(pydantic_object=ContractClause)

extract_prompt = """
请将以下合同文本拆解为结构化条款列表。

合同文本：
{contract_text}

要求：
1. 识别每个条款的标题和内容
2. 判断条款类型：payment / delivery / breach / confidentiality / termination / dispute / other
3. 保留原文，不要概括
4. 标注页码

输出 JSON 数组。
"""

def parse_contract(pdf_path):
    text = extract_text_from_pdf(pdf_path)
    response = llm.predict(extract_prompt.format(contract_text=text[:15000]))
    return json.loads(response)
```

**第二步：与模板库对比**

```python
# 构建合同模板向量库
template_kb = Chroma.from_documents(
    documents=[
        Document(page_content="付款条款标准模板：...", metadata={"type": "payment", "template": "标准采购"}),
        Document(page_content="违约金标准：不低于合同金额 20%...", metadata={"type": "breach", "template": "标准采购"}),
        # ...
    ],
    embedding=OpenAIEmbeddings()
)

def compare_with_template(clause, contract_type="采购"):
    # 检索对应类型的标准模板条款
    template_docs = template_kb.similarity_search(
        f"{contract_type} {clause['type']} {clause['title']}",
        filter={"type": clause["type"]},
        k=2
    )
    
    prompt = f"""
    你是一位资深企业法务。请对比以下条款与标准模板的差异。

    合同类型：{contract_type}
    当前条款（来自对方合同）：
    {clause['title']}: {clause['content']}

    我方标准模板参考：
    {format_docs(template_docs)}

    请输出 JSON：
    {{
      "deviation_type": "missing_risk / unfavorable / standard / better",
      "risk_level": "high / medium / low",
      "issues": ["问题1", "问题2"],
      "suggested_revision": "修改建议文本",
      "reason": "为什么这么改（对我方更有利）"
    }}
    """
    
    return llm.predict(prompt)
```

**第三步：风险条款识别**

```python
risk_keywords = {
    "high": [
        "无限责任", "放弃追偿", "自动续约", "单方解除",
        "管辖法院为对方所在地", "违约金低于 5%"
    ],
    "medium": [
        "口头协议有效", "知识产权归对方", "数据出境",
        "保密期过短", "验收标准模糊"
    ]
}

def risk_scan(clauses):
    risks = []
    for clause in clauses:
        for level, keywords in risk_keywords.items():
            for kw in keywords:
                if kw in clause["content"]:
                    risks.append({
                        "clause": clause["title"],
                        "keyword": kw,
                        "level": level,
                        "content": clause["content"][:200]
                    })
    return risks
```

**第四步：审阅报告生成**

```python
def generate_review_report(contract_name, clauses, comparisons, risks):
    prompt = f"""
    请为合同《{contract_name}》生成法务审阅报告。

    合同条款数：{len(clauses)}
    与模板对比异常：{len([c for c in comparisons if c['deviation'] != 'standard'])} 处
    风险标记：高风险 {len([r for r in risks if r['level']=='high'])} 处

    报告结构：
    1. 合同基本信息和交易对手
    2. 总体风险评估（高/中/低）
    3. 必须修改的条款清单（高风险）
    4. 建议优化的条款清单（中低风险）
    5. 缺失的我方保护条款
    6. 修改建议汇总表

    要求：专业法务语言，每处修改给出"原文"→"建议修改"→"理由"。
    """
    
    return llm.predict(prompt)
```

**第五步：Streamlit 审阅工作台**

```python
import streamlit as st

st.title("合同智能审阅助手")

uploaded = st.file_uploader("上传合同", type=["pdf", "docx"])
contract_type = st.selectbox("合同类型", ["采购", "销售", "NDA", "劳动合同", "技术服务"])

if uploaded:
    clauses = parse_contract(uploaded)
    
    # 左侧：条款列表
    with st.sidebar:
        for c in clauses:
            emoji = "🔴" if c["risk"] == "high" else "🟠" if c["risk"] == "medium" else "🟢"
            st.write(f"{emoji} {c['title']}")
    
    # 右侧：详细审阅
    selected = st.selectbox("查看条款", [c["title"] for c in clauses])
    clause = next(c for c in clauses if c["title"] == selected)
    
    st.write("**原文：**")
    st.info(clause["content"])
    
    comparison = compare_with_template(clause, contract_type)
    st.write("**风险分析：**")
    st.warning(comparison["issues"])
    
    st.write("**修改建议：**")
    st.success(comparison["suggested_revision"])
```

---

## 效果

在年审合同 2000+ 份的企业法务场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 单份合同初审时间 | 2-3h → 30min（Agent 预审后重点看） |
| 风险遗漏率 | 约 8%（人工） → 2%（Agent 标记 + 人复核） |
| 标准合同（NDA/采购模板） | 90% 可直接用 Agent 初稿 |
| 法务工作满意度 | 重复劳动减少，聚焦高价值谈判 |
| Junior 律师成长速度 | 通过对比 Agent 建议和自己判断，更快积累经验 |

---

## 要点

1. **模板库是核心资产** — Agent 的审查质量取决于模板库的完善程度。需要持续把"审过的优质合同"和"踩过的坑"沉淀为标准条款和风险清单。

2. **不能替代律师判断** — LLM 擅长识别标准风险，但商业条款的利弊权衡（比如接受低违约金换取更快账期）需要业务 context，必须由人决策。

3. **保密性** — 合同含商业机密，建议本地部署模型（Llama 3/Qwen 72B）+ 本地向量库，不上传云端。

4. **版本对比** — 合同通常改多轮，Agent 需要能对比 v1/v2/v3 的差异，识别对方"悄悄改回去"的条款。

---

## 扩展思路

- 接入工商信息 API，自动核查交易对手资质（失信、诉讼、注册资本）
- 合同履约监控：根据合同条款自动提醒付款节点、交付节点
- 电子签章集成：审完直接触发法大大/e签宝签署流程
- 多法域支持：跨境合同自动识别中美欧法律冲突条款

---

## 相关资源

- [Claude 长上下文 — 适合整份合同分析](https://docs.anthropic.com/en/docs/build-with-claude/long-context)
- [LangChain 结构化输出](https://python.langchain.com/docs/use_cases/extraction/)
- [国家市场监督管理总局合同示范文本](https://www.samr.gov.cn)
- [OpenAI 企业隐私政策](https://openai.com/enterprise-privacy)
