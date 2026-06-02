---
title: 用 RAG + LLM 搭建智能预问诊分诊 Agent
author: "@potato"
date: 2026-06
stack: LangChain, DeepSeek, ChromaDB, FastAPI, 医院HIS接口
difficulty: 专家
---

## 场景

三甲医院门诊量巨大，患者常见困扰：

1. **挂错号** — 不知道头痛该挂神经内科还是神经外科
2. **描述不清** — 面诊时 3 分钟说不明白症状，医生反复追问
3. **检查单不会看** — 拿到血常规/影像报告看不懂异常指标
4. **候诊焦虑** — 排队 2 小时，面诊 5 分钟，很多问题忘了问

目标是：**Agent 在患者挂号前收集症状、推荐科室、预问诊，并辅助解读检查报告**。

---

## 实现

### 架构

```
患者小程序 → 症状采集 → LLM 分诊推荐 → 挂号建议
               ↓              ↓
         预问诊报告      检查报告解读（RAG）
               ↓              ↓
         医生端提前查看    异常指标科普
```

### 核心实现

**第一步：症状采集对话**

```python
from langchain.memory import ConversationBufferMemory

class PreDiagnosisBot:
    def __init__(self):
        self.memory = ConversationBufferMemory()
        self.required_info = ["主诉", "持续时间", "加重/缓解因素", "既往病史", "过敏史"]
    
    def chat(self, user_input):
        prompt = f"""
        你是一位专业的预问诊护士。当前已收集信息：{self.memory.load_memory_variables()}
        
        患者说：{user_input}
        
        任务：
        1. 如果信息缺失，礼貌追问（每次最多追问 1 个）
        2. 如果信息足够，生成结构化预问诊报告
        3. 绝不给诊断结论，只说"建议就诊科室"和"可能需做检查"
        4. 遇到急危重症关键词（胸痛/昏迷/大出血），立即触发急救指引
        """
        
        response = llm.predict(prompt)
        self.memory.save_context({"input": user_input}, {"output": response})
        return response
```

**第二步：智能分诊**

```python
# 科室知识库
specialty_kb = Chroma.from_documents(
    documents=load_medical_textbooks(),
    embedding=OpenAIEmbeddings(),
    collection_name="specialty_guide"
)

def triage(symptoms):
    # RAG 检索相关分诊指南
    docs = specialty_kb.similarity_search(symptoms, k=3)
    
    prompt = f"""
    基于以下医学分诊指南：
    {format_docs(docs)}
    
    患者症状：{symptoms}
    
    请推荐最合适的 1-2 个就诊科室，并说明理由。
    如果症状不典型，建议"全科医学科"先做基础筛查。
    
    输出 JSON：
    {{"primary_dept": "主推荐科室", "secondary_dept": "次推荐", 
      "reason": "分诊理由", "suggested_tests": ["可能检查1", "检查2"]}}
    """
    
    return llm.predict(prompt)
```

**第三步：检查报告解读**

```python
def interpret_report(report_text, report_type="blood"):
    """用 RAG 解读检查报告，引用医学知识库"""
    
    # 检索该检查项目的正常参考值和临床意义
    docs = medical_kb.similarity_search(
        f"{report_type} 正常范围 临床意义",
        k=5
    )
    
    prompt = f"""
    你是一位医学科普作者。请解读以下检查报告。
    
    报告内容：
    {report_text}
    
    参考医学知识：
    {format_docs(docs)}
    
    要求：
    1. 用患者能听懂的大白话解释，不用拉丁文
    2. 标出异常指标，说明"高/低"意味着什么
    3. 说明"可能关联的疾病方向"（绝不给确诊）
    4. 建议"是否需要尽快复诊"
    5. 最后加一句"具体诊断请咨询主治医生"
    """
    
    return llm.predict(prompt)
```

**第四步：医生端预问诊报告**

```json
{
  "patient_id": "P12345",
  "chief_complaint": "头痛伴恶心 3 天",
  "duration": "3 天",
  "severity": "7/10",
  "associated_symptoms": ["恶心", "畏光"],
  "history": ["高血压病史 5 年"],
  "medications": ["氨氯地平"],
  "allergies": ["青霉素"],
  "triage_recommendation": "神经内科",
  "suggested_tests": ["头颅 CT", "血压监测"],
  "red_flags": ["血压 180/110", "建议优先排查脑血管意外"]
}
```

---

## 效果

在类似三甲医院门诊场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 挂错号率 | 约 15% → 5% |
| 医生首诊效率 | 平均问诊 8min → 5min（信息已预填） |
| 患者满意度 | 72% → 89%（候诊时焦虑感下降） |
| 检查单自助解读使用率 | 63%（减少护士咨询台压力） |

---

## 要点

1. **绝不替代医生诊断** — 所有输出必须包含"仅供参考，请遵医嘱"，系统设计要防止患者把 Agent 结论当确诊。

2. **急危重症识别是红线** — prompt 中必须包含胸痛/昏迷/大出血/呼吸困难等关键词的紧急处理流程，直接跳转急救电话而非继续对话。

3. **医学知识库必须权威** — RAG 数据来源建议用人民卫生出版社教材、UpToDate 临床顾问、医院内部诊疗规范。不要靠 LLM 的预训练知识。

4. **隐私合规** — 医疗数据属敏感个人信息，需通过等保三级、数据脱敏、本地部署向量库。

---

## 扩展思路

- 慢病管理：高血压/糖尿病患者定期随访，自动提醒用药和监测
- 用药助手：扫描药盒自动说明用法、禁忌、相互作用
- 术后康复指导：根据手术类型自动生成康复计划
- 多语言支持：服务外籍患者，自动翻译医学术语

---

## 相关资源

- [UpToDate 临床决策支持](https://www.uptodate.com)
- [人民卫生出版社教材](https://www.pmph.com)
- [医院信息系统 HIS 集成规范](https://www.nlhc.cn)
- [等保 2.0  healthcare 要求](https://www.miit.gov.cn)
