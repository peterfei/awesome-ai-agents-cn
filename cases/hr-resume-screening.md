---
title: 用 LLM + RAG 搭建智能招聘简历初筛 Agent
author: "@potato"
date: 2026-06
stack: Python, LangChain, GPT-4o, PostgreSQL, Streamlit
difficulty: 进阶
---

## 场景

成长型公司每月收到几百到上千份简历，HR 和用人部门面临以下问题：

1. **简历量与人力不匹配** — 1 名 HR 每天最多认真看 30-50 份简历，大量简历石沉大海
2. **标准不统一** — 不同 HR 对"匹配度"判断差异大，有人看学历，有人看项目
3. **JD 理解偏差** — 招聘启事写得笼统，HR 和用人部门对"核心要求"的理解不一致
4. **优秀人才被漏掉** — 非名校出身但有扎实项目经验的候选人，可能在初筛阶段就被学历门槛刷掉

该 Agent 目标：**根据 JD 和结构化评分标准，自动给每份简历打分并给出匹配理由，HR 只需复核高分简历**。

---

## 实现

### 整体架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   简历上传       │────▶│  结构化提取     │────▶│  匹配度评分      │
│  (PDF/Word/图片) │     │  (LLM/正则)     │     │  (多维度打分)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  HR 复核工作台     │
                    │  (Streamlit)       │
                    └─────────────────────┘
```

### 核心配置

**1. 简历结构化提取**

```python
# resume_parser.py
from openai import OpenAI
import pdfplumber

client = OpenAI()

def extract_text_from_pdf(path: str) -> str:
    text = ""
    with pdfplumber.open(path) as pdf:
        for page in pdf.pages:
            text += page.extract_text() or ""
    return text

RESUME_EXTRACTION_PROMPT = """从以下简历中提取结构化信息，输出 JSON：
{
  "name": "姓名",
  "education": [{"school": "", "degree": "", "major": "", "year": ""}],
  "work_experience": [{"company": "", "title": "", "duration": "", "highlights": [""] }],
  "projects": [{"name": "", "description": "", "tech_stack": [""]}],
  "skills": [""],
  "years_of_experience": float
}

简历内容：
{resume_text}

注意：
1. 如果信息缺失，值为 null
2. 技能列表去重，最多 20 个
3. 工作年限根据毕业时间和工作经历推算"""

def parse_resume(path: str) -> dict:
    text = extract_text_from_pdf(path)
    resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": RESUME_EXTRACTION_PROMPT.format(resume_text=text[:10000])}],
        response_format={"type": "json_object"}
    )
    return json.loads(resp.choices[0].message.content)
```

**2. 匹配度评分引擎**

```python
# scoring_engine.py
from openai import OpenAI

client = OpenAI()

SCORING_PROMPT = """你是一名专业招聘顾问。请根据以下 JD 和候选人简历，从 5 个维度打分并给出理由。

JD 描述：{jd}

候选人简历：{resume_json}

评分维度（每项 0-10 分）：
1. 技能匹配度：JD 要求的硬技能覆盖比例
2. 经验匹配度：工作年限、行业背景、岗位级别的匹配程度
3. 项目匹配度：过往项目与 JD 业务场景的相似度
4. 潜力评估：教育背景、学习能力、技术深度
5. 稳定性：工作经历的跳槽频率和每段时长

输出 JSON 格式：
{
  "skill_match": {"score": 8, "reason": "..."},
  "experience_match": {"score": 7, "reason": "..."},
  "project_match": {"score": 9, "reason": "..."},
  "potential": {"score": 7, "reason": "..."},
  "stability": {"score": 6, "reason": "..."},
  "total_score": 37,
  "recommendation": "强烈建议面试 / 建议面试 / 待定 / 不匹配",
  "concerns": ["可能的风险点1", "风险点2"]
}

要求：
1. 打分客观，不因为学历偏见而压低分数
2. 对于非名校但项目经验丰富的候选人，潜力分可以较高
3. 对频繁跳槽（平均<1年）要标红提醒"""

def score_resume(jd: str, resume: dict) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": SCORING_PROMPT.format(jd=jd, resume_json=json.dumps(resume, ensure_ascii=False))}],
        response_format={"type": "json_object"}
    )
    return json.loads(resp.choices[0].message.content)
```

**3. 批量处理与去重**

```python
# batch_processor.py
import os
from pathlib import Path

def process_batch(resume_dir: str, jd: str, db_session):
    results = []
    for path in Path(resume_dir).glob("*.pdf"):
        resume = parse_resume(str(path))
        score = score_resume(jd, resume)
        
        # 存入数据库
        record = ResumeScore(
            candidate_name=resume.get("name", "未知"),
            file_name=path.name,
            total_score=score["total_score"],
            recommendation=score["recommendation"],
            details=score,
            status="待复核"
        )
        db_session.add(record)
        results.append(record)
    
    db_session.commit()
    return results
```

**4. HR 复核工作台（Streamlit）**

```python
# app.py
import streamlit as st
from sqlalchemy.orm import Session

st.set_page_config(page_title="简历初筛助手", layout="wide")

st.title("📋 简历初筛工作台")

# 上传 JD
jd_text = st.text_area("粘贴 JD 描述", height=150)

# 上传简历压缩包
uploaded = st.file_uploader("上传简历文件夹（zip）", type=["zip"])

if uploaded and jd_text:
    with st.spinner("AI 正在分析..."):
        # 解压、解析、评分
        results = process_batch("/tmp/resumes", jd_text, db)
    
    # 按推荐度筛选
    filter_rec = st.selectbox("筛选", ["全部", "强烈建议面试", "建议面试", "待定", "不匹配"])
    
    for r in results:
        if filter_rec != "全部" and r.recommendation != filter_rec:
            continue
        
        col1, col2, col3 = st.columns([2, 1, 1])
        with col1:
            st.subheader(r.candidate_name)
            st.write(f"总分：**{r.total_score}** | 推荐：**{r.recommendation}**")
        with col2:
            st.json(r.details)
        with col3:
            if st.button("通过", key=f"pass_{r.id}"):
                update_status(r.id, "通过")
            if st.button("淘汰", key=f"reject_{r.id}"):
                update_status(r.id, "淘汰")
```

---

## 效果

在月均收 500+ 份简历的成长型公司中，预期效果：

| 指标 | 变化 |
|------|------|
| HR 初筛单份简历时间 | 3-5min → 30s（仅复核高分简历） |
| 日均处理简历量 | 40 份 → 150 份 |
| 初筛一致性 | 不同 HR 标准差异大 → AI 统一标准 |
| 用人部门面试通过率 | 约 20% → 35%（Agent 提前筛掉明显不匹配） |
| 漏掉优秀人才概率 | 约 15%（人工凭学历/公司名快速淘汰） → 降低 |

---

## 要点

1. **Prompt 必须对抗偏见** — 如果不加约束，LLM 容易给"大厂背景""名校学历"高分，给"自学转行"低分。建议在 prompt 中明确要求"不因学校排名而调整分数，只看技能和项目"。

2. **JD 要结构化输入** — 把 JD 中的"必备技能""加分项""必须经验"分开写，比扔给 LLM 一整段 JD 效果更好。可以维护一个 JD 模板库。

3. **隐私是红线** — 简历含大量个人信息。系统必须：加密存储、访问权限控制、定期清理未通过简历、符合《个人信息保护法》。不要在第三方 LLM API 中传输真实姓名+身份证号。

4. **Agent 是初筛不是终审** — 强烈建议面试/建议面试的简历仍需 HR 和用人部门复核。Agent 的作用是"把明显不匹配的筛掉，把可能匹配的推上来"。

---

## 延伸阅读

- [pdfplumber 文档](https://github.com/jsvine/pdfplumber)
- [Streamlit 官方文档](https://docs.streamlit.io/)
- [《个人信息保护法》企业合规要点](https://www.ppc.gov.cn/)

---

*Last updated: 2025-06-02*
