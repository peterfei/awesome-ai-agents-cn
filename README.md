# awesome-ai-agents-cn

> 中文 AI Agent 落地案例精选 —— 真实场景、真实代码、真实效果。

[![GitHub stars](https://img.shields.io/github/stars/peterfei/awesome-ai-agents-cn)](https://github.com/peterfei/awesome-ai-agents-cn)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 这是什么？

AI Agent 从 2024 年炒概念，到 2025 年真正落地。但中文社区缺少一个地方：

- 集中展示**真实跑起来的 Agent 案例**
- 每个案例回答：**解决什么问题？用了什么方案？效果如何？**
- 不止是链接列表，而是有深度、可复用的参考

这个项目就是补这个缺。

---

## 案例分类

### 🤖 客服与对话

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Dify 搭建电商客服自动回复 Agent](cases/customer-service-dify.md) | Dify + DeepSeek V3 + RAG | ★☆☆ |
| [用 Dify + LLM 搭建智能售后回访 Agent](cases/after-sales-followup.md) | Dify + GPT-4o-mini + RAG + 企业微信 | ★★☆ |

### 🔧 开发与工程

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Claude Code + MCP 做自动 Code Review](cases/code-review.md) | Claude Code + MCP + GitHub API | ★☆☆ |
| [用 Claude Code 自动生成 PR 描述](cases/pr-description.md) | Claude Code + Git + GitHub CLI | ★☆☆ |
| [用 Claude Code + Playwright 做自动化测试 Agent](cases/auto-testing.md) | Claude Code + Playwright + MCP + GitHub Actions | ★★☆ |
| [用 Claude Code 做自动化代码迁移 Agent](cases/code-migration.md) | Claude Code + AST + jscodeshift + TypeScript | ★★★ |

### 📊 数据与分析

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 LangGraph 搭建 Text-to-SQL 查询 Agent](cases/text-to-sql.md) | LangGraph + Claude + PostgreSQL | ★★★ |
| [用 Python + LLM 搭建 BI 报告自动生成 Agent](cases/bi-report-auto-generation.md) | Python + Pandas + LangChain + Streamlit + 钉钉 | ★★☆ |
| [用 LLM + BI 数据搭建数据异常监控与根因分析 Agent](cases/data-anomaly-root-cause.md) | Python + SQL + GPT-4o + Pandas + Streamlit + 钉钉 | ★★☆ |

### 📚 知识库与 RAG

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 LangChain + Ollama 搭建本地知识库 Agent](cases/local-knowledge-base.md) | LangChain + Ollama + ChromaDB + Qwen2.5 | ★★☆ |
| [用 LangChain + RAG 搭建学术文献综述与研究助手 Agent](cases/research-assistant.md) | LangChain + OpenAI + ChromaDB + Streamlit | ★★☆ |
| [用 RAG + LLM 搭建企业内部 IT Helpdesk 自动排障 Agent](cases/it-helpdesk-agent.md) | LangChain + GPT-4o + ChromaDB + FastAPI + Slack API | ★★☆ |

### 📝 内容与文档

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Coze 做公众号自动运营 Agent](cases/wechat-coze.md) | Coze + GPT-4o + RSS | ★☆☆ |

### 💼 办公效率

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Whisper + LLM 搭建会议记录自动整理 Agent](cases/meeting-minutes-agent.md) | Whisper + GPT-4o + Notion API + Python | ★☆☆ |
| [用 n8n + LLM 搭建邮件智能分类与自动回复 Agent](cases/email-auto-sort.md) | n8n + GPT-4o-mini + IMAP + Gmail API | ★☆☆ |
| [用 LLM + RAG 搭建智能招聘简历初筛 Agent](cases/hr-resume-screening.md) | Python + LangChain + GPT-4o + PostgreSQL + Streamlit | ★★☆ |

### 💰 销售与营销

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 n8n + GPT 做销售线索自动筛选与跟进 Agent](cases/sales-lead-qualification.md) | n8n + GPT-4o-mini + HubSpot API + 企业微信 | ★☆☆ |
| [用 Dify + 多模态模型做批量营销文案生成 Agent](cases/marketing-copy-generation.md) | Dify + GPT-4o + Kimi + 飞书多维表格 | ★★☆ |
| [用 Python + Playwright 搭建竞品价格监控与策略分析 Agent](cases/competitor-price-monitor.md) | Python + Playwright + GPT-4o-mini + PostgreSQL + 钉钉 | ★★☆ |

### ⚙️ 运营与流程自动化

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 n8n + ERP API 搭建订单异常自动处理 Agent](cases/order-processing-agent.md) | n8n + PostgreSQL + 企业微信 + ERP API | ★★☆ |
| [用 Python + LLM 搭建库存预警与智能补货建议 Agent](cases/inventory-alert-agent.md) | Python + Pandas + GPT-4o-mini + PostgreSQL + 钉钉 | ★★☆ |

### 🔒 安全与合规

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 AI 做自动化代码安全审计 Agent](cases/security-audit.md) | Semgrep + Claude Code + MCP + OWASP | ★★☆ |

### 🏭 行业垂直

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 LLM + RAG 搭建财报智能分析 Agent](cases/financial-report-analysis.md) | LangChain + Claude Sonnet + ChromaDB + Streamlit | ★★☆ |
| [用多模态 LLM 搭建作业自动批改与学情分析 Agent](cases/education-homework-grading.md) | GPT-4o Vision + Python + FastAPI + PostgreSQL | ★★☆ |
| [用 RAG + LLM 搭建智能预问诊分诊 Agent](cases/medical-pre-diagnosis.md) | LangChain + DeepSeek + ChromaDB + FastAPI + HIS | ★★★ |
| [用 RAG + LLM 搭建合同智能审查 Agent](cases/legal-contract-review.md) | LangChain + Claude Sonnet + ChromaDB + Streamlit | ★★★ |
| [用视觉大模型 + IoT 搭建产线质检 Agent](cases/manufacturing-quality-inspection.md) | YOLOv8 + GPT-4o Vision + Edge Device + MQTT | ★★★ |
| [用 RAG + 大模型搭建政务服务热线智能答复 Agent](cases/government-service-hotline.md) | Dify + DeepSeek + RAG + PostgreSQL + 政务中台 | ★★☆ |

### 🤝 多 Agent 协作

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 LangGraph 搭建多 Agent 协作内容生产流水线](cases/multi-agent-content-factory.md) | LangGraph + Claude Sonnet + Tavily + DALL-E + Airtable | ★★★ |

### 🚀 运维与部署

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Claude Code + MCP 做日志分析故障排查 Agent](cases/log-analysis.md) | Claude Code + MCP + Shell + Kubernetes | ★★☆ |

---

## 技术栈索引

| 框架/工具 | 相关案例 |
|-----------|----------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) | Code Review Agent, PR 描述生成, 自动化测试 Agent, 代码迁移 Agent |
| [Dify](https://docs.dify.ai) | 电商客服 Agent, 营销文案生成 Agent, 售后回访 Agent |
| [MCP](https://modelcontextprotocol.io) (Model Context Protocol) | Code Review Agent, 日志分析 Agent, 自动化测试 Agent, 安全审计 Agent |
| [Semgrep](https://semgrep.dev) | 安全审计 Agent |
| [OWASP](https://owasp.org) | 安全审计 Agent |
| [Playwright](https://playwright.dev) | 自动化测试 Agent, 竞品监控 Agent |
| [Shell](https://en.wikipedia.org/wiki/Shell_script) | 日志分析 Agent |
| [Kubernetes](https://kubernetes.io) | 日志分析 Agent |
| [Coze](https://www.coze.com) | 公众号运营 Agent |
| [LangGraph](https://langchain-ai.github.io/langgraph/) | Text-to-SQL Agent, 多 Agent 内容工厂 |
| [n8n](https://n8n.io) | 邮件分类 Agent, 销售线索筛选 Agent, 订单处理 Agent |
| [Streamlit](https://streamlit.io) | 财报分析 Agent, 合同审查 Agent, BI 报告 Agent, 文献综述 Agent, 数据异常分析 Agent, 招聘初筛 Agent |
| DeepSeek | 电商客服 Agent, 政务热线 Agent |
| GPT-4o | 公众号运营 Agent, 会议记录 Agent, 邮件分类 Agent, 营销文案 Agent, IT Helpdesk Agent, 数据异常分析 Agent, 招聘初筛 Agent |
| Claude Sonnet | Text-to-SQL Agent, 合同审查 Agent, 财报分析 Agent |
| [RAG](https://python.langchain.com/docs/use_cases/question_answering/) | 电商客服 Agent, 本地知识库 Agent, 政务热线 Agent, 合同审查 Agent, 预问诊 Agent, 售后回访 Agent, 文献综述 Agent |
| [Ollama](https://ollama.com) | 本地知识库 Agent |
| [LangChain](https://www.langchain.com) | 本地知识库 Agent, 合同审查 Agent, 财报分析 Agent, BI 报告 Agent, 文献综述 Agent, IT Helpdesk Agent, 招聘初筛 Agent |
| [ChromaDB](https://www.trychroma.com) | 本地知识库 Agent, 合同审查 Agent, 财报分析 Agent, 文献综述 Agent, IT Helpdesk Agent |
| [Qwen2.5](https://github.com/QwenLM/Qwen2.5) | 本地知识库 Agent |
| PostgreSQL | Text-to-SQL Agent, 订单处理 Agent, 库存预警 Agent, 竞品监控 Agent, 数据异常分析 Agent, 招聘初筛 Agent |
| [Whisper](https://github.com/openai/whisper) | 会议记录 Agent |
| [Notion API](https://developers.notion.com) | 会议记录 Agent |
| [YOLO](https://docs.ultralytics.com) | 产线质检 Agent |
| [MQTT](https://mqtt.org) | 产线质检 Agent |
| [Tavily](https://tavily.com) | 多 Agent 内容工厂 |
| [DALL-E](https://platform.openai.com/docs/guides/images) | 多 Agent 内容工厂 |
| [FastAPI](https://fastapi.tiangolo.com) | 作业批改 Agent, 预问诊 Agent, IT Helpdesk Agent |
| [钉钉/飞书/企微](https://open.dingtalk.com) | 库存预警 Agent, 邮件分类 Agent, 订单处理 Agent, 销售线索 Agent, 售后回访 Agent, BI 报告 Agent, 竞品监控 Agent, 数据异常分析 Agent |

---

## 贡献

欢迎提交案例。要求：

1. 你**亲自用过**或**见过真实效果**（不是读文档后的理论总结）
2. 按模板填写（见 [CONTRIBUTING.md](CONTRIBUTING.md)）
3. 包含效果数据（量化优先，定性也可）

---

## 路线图

- [x] 开发工程系列（Code Review、PR 描述、自动化测试、代码迁移）
- [x] 客服场景系列（Dify 电商客服）
- [x] 💼 办公效率系列（会议记录、邮件分类）
- [x] 销售营销系列（线索筛选、营销文案）
- [x] 运营自动化系列（订单处理、库存预警）
- [x] 内容运营系列（Coze 公众号运营、多 Agent 内容工厂）
- [x] 数据分析系列（Text-to-SQL）
- [x] 运维部署系列（日志分析）
- [x] 知识库与 RAG 系列（本地知识库）
- [x] 安全合规系列（代码安全审计）
- [x] 行业垂直系列（金融财报、教育作业、医疗预问诊、法律合同、制造质检、政务热线）
- [x] 多 Agent 协作系列（内容生产流水线）
- [ ] 季度趋势分析

---

## License

MIT
