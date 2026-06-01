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

### 🔧 开发与工程

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Claude Code + MCP 做自动 Code Review](cases/code-review.md) | Claude Code + MCP + GitHub API | ★☆☆ |
| [用 Claude Code 自动生成 PR 描述](cases/pr-description.md) | Claude Code + Git + GitHub CLI | ★☆☆ |

### 📊 数据与分析

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 LangGraph 搭建 Text-to-SQL 查询 Agent](cases/text-to-sql.md) | LangGraph + Claude + PostgreSQL | ★★★ |

### 📚 知识库与 RAG

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 LangChain + Ollama 搭建本地知识库 Agent](cases/local-knowledge-base.md) | LangChain + Ollama + ChromaDB + Qwen2.5 | ★★☆ |

### 📝 内容与文档

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Coze 做公众号自动运营 Agent](cases/wechat-coze.md) | Coze + GPT-4o + RSS | ★☆☆ |

### 🚀 运维与部署

| 案例 | 技术栈 | 难度 |
|------|--------|------|
| [用 Claude Code + MCP 做日志分析故障排查 Agent](cases/log-analysis.md) | Claude Code + MCP + Shell + Kubernetes | ★★☆ |

---

## 技术栈索引

| 框架/工具 | 相关案例 |
|-----------|----------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) | Code Review Agent, PR 描述生成 |
| [MCP](https://modelcontextprotocol.io) (Model Context Protocol) | Code Review Agent, 日志分析 Agent |
| [Shell](https://en.wikipedia.org/wiki/Shell_script) | 日志分析 Agent |
| [Kubernetes](https://kubernetes.io) | 日志分析 Agent |
| [Dify](https://docs.dify.ai) | 电商客服 Agent |
| [Coze](https://www.coze.com) | 公众号运营 Agent |
| [LangGraph](https://langchain-ai.github.io/langgraph/) | Text-to-SQL Agent |
| DeepSeek | 电商客服 Agent |
| GPT-4o | 公众号运营 Agent |
| Claude Sonnet | Text-to-SQL Agent |
| [RAG](https://python.langchain.com/docs/use_cases/question_answering/) | 电商客服 Agent, 本地知识库 Agent |
| [Ollama](https://ollama.com) | 本地知识库 Agent |
| [LangChain](https://www.langchain.com) | 本地知识库 Agent |
| [ChromaDB](https://www.trychroma.com) | 本地知识库 Agent |
| [Qwen2.5](https://github.com/QwenLM/Qwen2.5) | 本地知识库 Agent |
| PostgreSQL | Text-to-SQL Agent |

---

## 贡献

欢迎提交案例。要求：

1. 你**亲自用过**或**见过真实效果**（不是读文档后的理论总结）
2. 按模板填写（见 [CONTRIBUTING.md](CONTRIBUTING.md)）
3. 包含效果数据（量化优先，定性也可）

---

## 路线图

- [x] 开发工程系列（Code Review、PR 描述）
- [x] 客服场景系列（Dify）
- [x] 内容运营系列（Coze）
- [x] 数据分析系列（Text-to-SQL）
- [x] 运维部署系列（日志分析）
- [x] 知识库与 RAG 系列（本地知识库）
- [ ] 季度趋势分析

---

## License

MIT
