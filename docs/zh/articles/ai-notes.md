---
title: AI 学习笔记
date: 2026-10-26
description: 关于大语言模型、AI Agent 与 MCP 协议的学习记录
tags: [AI, 机器学习, 技术笔记]
---

# AI 学习笔记 🤖

记录在学习人工智能和大语言模型过程中的思考与笔记。

---

## 什么是大语言模型？

大语言模型（LLM, Large Language Model）是基于 Transformer 架构、在海量文本数据上训练而成的深度学习模型。它能够理解并生成自然语言，完成从对话到代码生成的各种任务。

### 核心概念

| 概念 | 说明 |
|------|------|
| **Token** | 文本的最小处理单元，一个词可以被拆分为多个 token |
| **上下文窗口** | 模型一次能处理的最大 token 数量 |
| **Temperature** | 控制生成文本的随机性，值越高越富有创意 |
| **API** | 应用编程接口，通过 HTTP 请求调用模型能力 |

---

## AI Agent 与 MCP 协议

### AI Agent

AI Agent 是能够自主感知环境、做出决策并执行行动的智能系统。典型的 Agent 工作流：

```
感知 → 思考 → 决策 → 行动 → 反馈
```

### MCP 协议

MCP（Model Context Protocol）是 Anthropic 提出的开放协议，用于标准化 AI 模型与外部工具、数据源之间的交互。

**核心优势：**
- 统一的工具调用接口
- 安全的沙箱执行环境
- 跨平台互操作性

---

## 实践项目

### 使用 docmd 构建 AI 文档站

docmd 原生支持 MCP 服务器集成，让 AI 代理能够：
- 读取文档上下文
- 回答基于文档的问题
- 自动生成代码示例

```bash
# 启动 docmd MCP 服务器
npx docmd mcp
```

---

## 学习资源

- [Anthropic Claude 文档](https://docs.anthropic.com)
- [MCP 官方规范](https://modelcontextprotocol.io)
- [Hugging Face 模型库](https://huggingface.co)

---

> 💡 **持续更新中** — 随着学习的深入，这里会不断补充新的内容。欢迎通过 GitHub 提交 PR 或 Issue 交流讨论。
