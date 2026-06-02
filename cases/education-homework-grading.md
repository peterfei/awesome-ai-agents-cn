---
title: 用多模态 LLM 搭建作业自动批改与学情分析 Agent
author: "@potato"
date: 2026-06
stack: GPT-4o Vision, Python, FastAPI, PostgreSQL
difficulty: 进阶
---

## 场景

K12 教育培训机构老师每天批改 90+ 份作业，痛点：

1. **批改量大** — 人工批改 1 份 3-5 分钟，反馈滞后到第二天
2. **学情统计难** — 全班哪些知识点薄弱？靠经验猜
3. **个性化辅导缺位** — 统一讲评无法针对每个学生的错题精准辅导

目标是：**学生拍照上传作业，Agent 秒级批改并生成个性化错题解析+班级学情报告**。

---

## 实现

### 架构

```
学生拍照 → 多模态识别 → 自动批改 → 个性化解析
                              ↓
                        教师端学情报告
```

### 核心实现

**第一步：题目识别**

```python
import base64
from openai import OpenAI

client = OpenAI()

def recognize_homework(image_path):
    with open(image_path, "rb") as f:
        b64 = base64.b64encode(f.read()).decode()
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": """
                识别作业图片，输出 JSON：
                {"questions": [{"id": "题号", "type": "choice/fill/calc", 
                   "content": "题目", "student_answer": "答案"}]}
                """},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}}
            ]
        }],
        response_format={"type": "json_object"}
    )
    return json.loads(response.choices[0].message.content)
```

**第二步：自动批改**

```python
def grade_question(q, student_ans, answer_key):
    if q["type"] == "choice":
        correct = student_ans.upper().strip() == answer_key.upper().strip()
        return {"correct": correct, "score": q["points"] if correct else 0}
    
    # 计算/解答题用 LLM 判断
    prompt = f"""
    题目：{q['content']}
    标准答案：{answer_key}
    学生答案：{student_ans}
    满分：{q['points']}
    
    请输出 JSON：
    {{"correct": true/false, "score": 0-{q['points']}, 
      "feedback": "点评", "knowledge_point": "知识点"}}
    """
    return call_llm(prompt)
```

**第三步：个性化解析**

```python
def generate_explanation(student_id, wrong_qs, history_weaknesses):
    prompt = f"""
    你是亲切的老师。学生薄弱点：{history_weaknesses}
    本次错题：{wrong_qs}
    
    要求：
    1. 语气亲切，指出错误根因
    2. 给出正确解法（关键步骤高亮）
    3. 推荐 1-2 道同类基础题
    4. Markdown 格式
    """
    return call_llm(prompt)
```

**第四步：班级学情报告**

```python
def class_report(class_id, date):
    stats = db.query("""
        SELECT knowledge_point, 
               AVG(score) as avg_score,
               SUM(CASE WHEN score=0 THEN 1 ELSE 0 END)*1.0/COUNT(*) as error_rate
        FROM homework_records 
        WHERE class_id=%s AND date=%s
        GROUP BY knowledge_point
    """, (class_id, date))
    
    prompt = f"""
    班级数据：{stats.to_markdown()}
    
    请生成学情报告：
    1. 整体得分率
    2. 错误率最高的 3 个知识点
    3. 重点关注学生名单
    4. 明天讲评建议
    5. 分层作业建议
    """
    return call_llm(prompt)
```

---

## 效果

在 500 人规模的 K12 教培场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 单份批改时间 | 人工 3-5min → AI 15s |
| 反馈延迟 | 次日 → 拍照后 30s |
| 老师日工作量 | 4-5h → 1h（重点看学情报告+处理疑难） |
| 学生错题重做率 | 45% → 78%（即时反馈激发重做） |
| 班级平均分 | 持平（但高分段人数增加 12%） |

---

## 要点

1. **手写识别准确率** — 潦草字迹、涂改、非标准符号容易识别错误。建议学生在框内书写，Agent 对不确定答案标[?]请老师复核。

2. **解答题按步骤给分** — LLM 批改计算题结果对但步骤错时，需要评分细则指导。建议先录入标准解题步骤作为参考。

3. **隐私保护** — 学生作业含个人信息，图片存储需加密，符合教育数据安全规范。

4. **避免直接给答案** — 错题解析应给"引导"而非"答案"，否则学生直接抄。prompt 中加规则"只给思路提示，不给最终数字答案"。

---

## 扩展思路

- 接入 OCR 专用模型（如 Mathpix）提升数学公式识别率
- 长期学情追踪：自动生成每个学生的知识图谱薄弱点热力图
- 自适应练习：根据错题自动推送难度递进的同类题目
- 家长端周报：每周自动推送孩子学习情况摘要

---

## 相关资源

- [GPT-4o Vision API](https://platform.openai.com/docs/guides/vision)
- [Mathpix OCR — 数学公式识别](https://mathpix.com)
- [FastAPI 官方文档](https://fastapi.tiangolo.com)
