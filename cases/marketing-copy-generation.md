---
title: 用 Dify + 多模态模型做批量营销文案生成 Agent
author: "@potato"
date: 2026-06
stack: Dify, GPT-4o, Kimi, 飞书多维表格, Python
difficulty: 进阶
---

## 场景

电商/品牌团队每月需要产出大量营销素材：

- 小红书/抖音文案：100+ 条
- 商品详情页卖点：50+ 个 SKU
- 朋友圈/社群海报文案：每天 3-5 条
- 不同平台风格各异（小红书的emoji和语气 vs 淘宝的功能性描述）

纯人工写：3 个内容运营满负荷工作。外包写：质量不稳定、调性不统一。

目标是：**Agent 根据商品信息和平台要求，批量生成符合各平台调性的营销文案，人工只做审核和微调**。

---

## 实现

### 架构

```
商品信息表（飞书） → Dify Workflow → ① 小红书文案
                                 → ② 抖音脚本
                                 → ③ 淘宝详情页
                                 → ④ 朋友圈文案
                      → 人工审核飞书表格 → 导出发布
```

### 核心配置

**第一步：商品信息标准化输入**

飞书多维表格字段：

| 字段 | 示例 |
|------|------|
| 商品名 | 无线降噪耳机 Pro |
| 核心卖点 | 40dB降噪, 30小时续航, 蓝牙5.3 |
| 目标人群 | 25-35岁上班族, 通勤族 |
| 价格带 | 299元（中端） |
| 平台 | 小红书/抖音/淘宝/朋友圈 |
| 参考竞品 | AirPods Pro, 华为FreeBuds |

**第二步：Dify Workflow 多分支生成**

```yaml
# Dify Workflow 配置
workflow:
  name: 营销文案工厂
  
  nodes:
    - type: start
      inputs: [商品名, 核心卖点, 目标人群, 价格带, 平台, 参考竞品]
    
    - type: if-else
      condition: platform
      branches:
        - xiaohongshu:
            - type: llm
              model: gpt-4o
              prompt: |
                你是一个小红书头部种草博主。请为以下商品写一篇种草笔记。
                
                商品：{{商品名}}
                卖点：{{核心卖点}}
                目标人群：{{目标人群}}
                价格：{{价格带}}
                
                要求：
                1. 标题带emoji，有悬念或数字
                2. 正文分 3-5 段，每段 2-3 行
                3. 语气像闺蜜分享，避免广告感
                4. 必带话题标签 5-8 个
                5. 结尾引导互动（"姐妹们觉得值吗？"）
                6. 总字数 200-400 字
                
                输出格式：标题 | 正文 | 话题标签
                
        - taobao:
            - type: llm
              model: gpt-4o
              prompt: |
                你是一个资深电商文案。请为以下商品写淘宝详情页卖点。
                
                商品：{{商品名}}
                卖点：{{核心卖点}}
                价格：{{价格带}}
                
                要求：
                1. 提炼 5 个核心卖点，每个卖点配一句 Slogan
                2. 痛点场景化（"通勤时地铁太吵？"）
                3. 卖点数据化（"降噪深度达 40dB"）
                4. 最后加 1 句信任背书（"30天无理由退换"）
                
                输出格式：
                卖点1：标题 | 描述
                ...
                
        - douyin:
            - type: llm
              model: gpt-4o
              prompt: |
                你是一个抖音带货脚本策划。请写一个 30-45 秒的短视频脚本。
                
                商品：{{商品名}}
                卖点：{{核心卖点}}
                价格：{{价格带}}
                
                要求：
                1. 前 3 秒必须有钩子（"为什么我说..."）
                2. 中间展示核心卖点 + 使用场景
                3. 结尾引导转化（"左下角链接，今天限时优惠"）
                4. 标注画面切换和字幕风格
                
                输出格式：时间轴 | 画面 | 台词/字幕
```

**第三步：质量评估节点**

```python
def quality_check(text, platform):
    """用 LLM 做二次审核，检查文案质量"""
    
    check_prompt = f"""
    请审核以下{platform}文案，输出 JSON：
    
    {{
      "pass": true/false,
      "score": 1-10,
      "issues": ["问题1", "问题2"],
      "improved_version": "改进后的文案"
    }}
    
    审核标准：
    - 是否包含违禁词（最、第一、绝对等极限词）
    - 是否符合平台调性
    - 是否有明显语法错误
    - 是否信息准确（不要编造参数）
    """
    
    return call_llm(check_prompt)
```

**第四步：飞书表格回写**

```python
# Python 脚本批量写入飞书
from lark_oapi import Client

client = Client(app_id="", app_secret="")

records = []
for product in products:
    for platform in ["小红书", "抖音", "淘宝", "朋友圈"]:
        copy = generate_copy(product, platform)
        qc = quality_check(copy, platform)
        
        records.append({
            "商品名": product["name"],
            "平台": platform,
            "文案内容": copy,
            "质量分": qc["score"],
            "审核状态": "待审核" if qc["pass"] else "需修改",
        })

# 批量写入飞书多维表格
client.bitable.v1.app_table_record.batch_create(records)
```

---

## 效果

在月销 300 万量级的 DTC 品牌场景中，预期效果：

| 指标 | 变化 |
|------|------|
| 单 SKU 文案产出时间 | 2h → 5min（人工审核时间） |
| 月产出文案量 | 200 条 → 800 条 |
| 文案过审率 | 第一次 72% → 优化 prompt 后 88% |
| 内容运营人效 | 人均管理 10 个 SKU → 30 个 SKU |
| 小红书互动率 | 2.1% → 3.4%（AI 更懂平台话术） |
| 极限词违规率 | 人工偶尔漏 → AI 100% 拦截 |

**翻车与修复**：
- 问题：Agent 把耳机续航"30小时"写成"300小时"
- 修复：在 prompt 中加规则"所有数字必须与输入信息一致，不要夸张"
- 问题：小红书文案"AI 味"重，像广告不像种草
- 修复：加入 few-shot 示例（3 篇真人写的爆款笔记作为参考）

---

## 要点

1. **Few-shot 比调 prompt 管用** — 给 LLM 看 3-5 篇你们团队写过的高赞文案，比写 500 字 prompt 描述"调性"有效得多。

2. **质量审核必须双层** — LLM 生成 → LLM 自检 → 人工终审。第一层检查违禁词和事实错误，第二层把控品牌调性。

3. **平台差异化是核心** — 同一商品，小红书的"闺蜜分享"、抖音的"钩子+转化"、淘宝的"功能参数"完全不同。不要试图用一个 prompt 通吃所有平台。

4. **数据闭环优化** — 把各平台文案的真实表现（点击率、互动率、转化率）回流到系统中，用来优化生成策略。高转化文案的风格要作为正向 few-shot 保留。

---

## 扩展思路

- 接入即梦/Midjourney API，根据文案自动生成配图提示词并产图
- A/B 测试自动化：同一商品生成 3 版文案，自动投放测试，优胜者批量推广
- 热点追踪：接入微博热榜/抖音热点，自动把热点梗融入文案
- 多语言出海：中文文案 → 自动翻译+本地化 → 适配欧美/东南亚市场

---

## 相关资源

- [Dify Workflow 文档](https://docs.dify.ai/guides/workflow)
- [飞书多维表格 API](https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/reference/bitable-v1/app-table-record/create)
- [小红书社区规范](https://www.xiaohongshu.com/protocols/copyright)
- [广告法违禁词库](https://www.samr.gov.cn/zw/zfxxgk/fdzdgknr/fgs/202001/t20200103_310376.html)
