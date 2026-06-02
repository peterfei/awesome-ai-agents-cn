---
title: 用 Whisper + LLM 搭建会议记录自动整理 Agent
author: "@potato"
date: 2026-06
stack: Whisper, GPT-4o, Notion API, Python, WebSocket
difficulty: 入门
---

## 场景

团队每天开 3-5 个会（站会、评审、复盘），会后整理会议纪要是个苦差事：

1. **记录不完整** — 人工记录容易漏掉决策点和待办
2. **整理耗时** — 1 小时的会议，整理纪要需要 30-45 分钟
3. **分发困难** — 纪要散落在各人文档里，待办没人跟进

目标是：**会议结束后 2 分钟内自动生成结构化纪要，并同步到 Notion 创建待办**。

---

## 实现

### 架构

```
会议语音/录屏 → Whisper 实时转录 → LLM 结构化整理 → Notion 写入
                                    ↓
                              飞书/企微推送摘要
```

### 核心实现

**第一步：实时语音转录**

```python
import whisper
import pyaudio
import numpy as np

model = whisper.load_model("base")

# 实时音频流处理
def audio_callback(in_data, frame_count, time_info, status):
    audio_np = np.frombuffer(in_data, np.int16).astype(np.float32) / 32768.0
    result = model.transcribe(audio_np, language="zh")
    return result["text"]
```

生产环境建议用 `whisper.cpp` 或云 API（通义听悟、讯飞）降低本地 GPU 压力。

**第二步：LLM 结构化整理**

```python
from openai import OpenAI

client = OpenAI()

prompt = """
你是一位专业的会议助理。请将以下会议转录文本整理成结构化纪要。

转录文本：
{transcript}

请输出以下格式：

## 会议信息
- 时间：
- 参会人：
- 会议主题：

## 核心结论（3 条以内）
1.
2.
3.

## 决策事项
| 决策 | 决策人 | 背景 |
|------|--------|------|
| ...  | ...    | ...  |

## 待办事项
| 事项 | 负责人 | 截止日期 | 优先级 |
|------|--------|----------|--------|
| ...  | ...    | ...      | P0/P1  |

## 风险提示
- 如果有哪些事项听起来有风险或需要关注，请列出

注意：
- 如果转录中有听不清的地方，标注[?]
- 不要编造未提及的内容
- 待办必须有明确的负责人（如果听不清标[待确认]）
"""

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": prompt}]
)
```

**第三步：自动写入 Notion**

```python
from notion_client import Client

notion = Client(auth=os.environ["NOTION_TOKEN"])

# 创建会议纪要页面
page = notion.pages.create(
    parent={"database_id": "会议纪要数据库ID"},
    properties={
        "标题": {"title": [{"text": {"content": title}}]},
        "日期": {"date": {"start": meeting_date}},
        "参会人": {"multi_select": [{"name": p} for p in participants]},
    },
    children=[...]  # 结构化内容块
)

# 同步创建待办事项到任务看板
for todo in todos:
    notion.pages.create(
        parent={"database_id": "待办数据库ID"},
        properties={
            "任务": {"title": [{"text": {"content": todo["task"]}}]},
            "负责人": {"people": [{"id": user_id}]},
            "截止日期": {"date": {"start": todo["due"]}},
            "来源会议": {"relation": [{"id": page["id"]}]},
        }
    )
```

**第四步：飞书/企微推送**

```python
# 发送会议摘要到群
summary = f"""
📋 {title} 纪要已生成

🎯 核心结论：
{conclusions}

✅ 待办 {len(todos)} 项：
{todo_list}

🔗 完整纪要：{notion_url}
"""

send_feishu_message(group_chat_id, summary)
```

---

## 效果

在 60 人左右的产品技术团队场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 纪要整理时间 | 45min → 2min |
| 待办遗漏率 | 约 25% → 5% |
| 待办按时完成率 | 60% → 82% |
| 人工review时间 | 5min/篇（检查准确性） |
| 参会人满意度 | 87% |

**典型场景**：评审会后，相关同学 2 分钟内收到待办清单，其中待办自动关联到 Notion 看板，负责人收到飞书提醒。

---

## 要点

1. **转录质量决定上限** — 环境噪音、多人同时说话会导致转录错乱。建议要求会议中发言人清晰报出自己名字（"我是张三，我认为..."），或者给每人配独立麦克风。

2. **区分闲聊和有效内容** — 会议前后 5 分钟的闲聊不需要整理。可以用规则过滤（"开始"关键词前、"结束"关键词后），或者 LLM 判断内容相关性。

3. **待办的截止日期必须人工确认** — Agent 经常把"下周"理解错误，或把模糊表述硬凑成日期。待办中的日期建议标[待确认]，由负责人自行修改。

4. **敏感会议不要自动同步** — HR 面谈、薪酬讨论等需要设置白名单/黑名单，避免隐私泄露。

---

## 扩展思路

- 接入日历系统自动提取参会人名单和会议主题
- 会后 24 小时自动追问待办进度（飞书机器人）
- 长期积累后做会议效率分析：平均会议时长、决策率、待办完成率
- 多语言会议支持（中英文混合转录 + 纪要生成）

---

## 相关资源

- [OpenAI Whisper 官方文档](https://github.com/openai/whisper)
- [Notion API 文档](https://developers.notion.com)
- [通义听悟 — 阿里云语音转录](https://tingwu.aliyun.com)
- [飞书机器人开发指南](https://open.feishu.cn)
