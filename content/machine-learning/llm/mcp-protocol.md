---
title: MCP协议（Model Context Protocol）
date: 2026-04-14
description: Model Context Protocol（MCP）是一种开放标准，用于标准化AI代理与外部工具、数据源和服务的连接方式
tags:
  - llm
  - ai-agent
  - mcp
  - tool-calling
draft: false
permalink:
---

## 概述

Model Context Protocol（MCP）是由Anthropic于2024年11月提出的开放协议，旨在标准化AI代理如何连接外部工具、数据源和服务。[^1]

MCP被称为AI集成的"USB-C接口"——一种任何代理都可以用来与任何MCP兼容服务器交互的单一接口。在MCP之前，每个AI助手都有自己专有的插件或工具格式，而MCP统一了连接层，允许单个服务器实现被Claude、任何未来的MCP兼容代理或构建在LLM之上的自定义代理使用。

### MCP vs Function Calling

| 特性 | Function Calling | MCP |
|------|-----------------|-----|
| 通信模式 | 请求-响应 | 持续对话式 |
| 标准化程度 | 厂商特定 | 开放统一 |
| 上下文保持 | 每次独立 | 协议层支持 |
| 工具发现 | 手动配置 | 自动发现机制 |
| 适用场景 | 简单API调用 | 多工具、实时更新 |

## 架构

### 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                     AI Agent (MCP Client)                   │
└─────────────────────────────────────────────────────────────┘
                              ↕ MCP Protocol (JSON-RPC over stdio/SSE/HTTP)
┌─────────────────────────────────────────────────────────────┐
│                      MCP Server                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐    │
│  │   Tools     │  │  Resources  │  │     Prompts     │    │
│  │  (函数调用)  │  │  (数据资源)  │  │    (提示模板)    │    │
│  └─────────────┘  └─────────────┘  └─────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 三种核心能力

| 能力 | 描述 | 示例 |
|------|------|------|
| **Tools** | 代理可以调用的函数 | `search_products`、`create_invoice` |
| **Resources** | 代理可以读取的数据 | 文件系统、数据库、API响应 |
| **Prompts** | 可重用的提示模板 | 代码审查模板、文档生成模板 |

### 传输协议

MCP支持多种传输方式：

| 传输方式 | 适用场景 | 特点 |
|----------|----------|------|
| **stdio** | 本地服务器 | 简单、最适合本地开发 |
| **Streamable HTTP** | 本地/远程服务器 | 2025年6月后推荐 |
| **Server-Sent Events** | 遗留支持 | 已弃用 |

## 工具设计原则

### 效率优化

Cloudflare在2026年2月的MCP实现中提出了一个关键洞察：**具有广泛能力的少量工具优于大量窄工具**。[^2]

**问题示例**：将2,500个API端点暴露为2,500个工具将消耗超过100万个token的上下文。

**Code Mode模式**：服务器只暴露少量工具（如`search()`探索API规范，`execute()`执行请求），代理发送代码在服务器沙箱中运行。

### 工具分组策略

```json
{
  "name": "database_ops",
  "description": "Perform database operations including CRUD on tables",
  "inputSchema": {
    "type": "object",
    "properties": {
      "action": {
        "type": "string",
        "enum": ["select", "insert", "update", "delete"],
        "description": "The operation to perform"
      },
      "table": {
        "type": "string",
        "description": "Target table name"
      },
      "filter": {
        "type": "object",
        "description": "WHERE clause conditions"
      }
    }
  }
}
```

## 应用场景

### 文档站点

MCP服务器使代理能够：
- 程序化搜索内容
- 按slug检索特定文章、文档或词汇表条目

```json
{
  "name": "search_docs",
  "description": "Search the documentation. Returns a list of matching pages.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search query"
      }
    },
    "required": ["query"]
  }
}
```

### 企业系统集成

```python
# MCP服务器暴露企业系统能力
class EnterpriseMCPServer:
    def __init__(self):
        self.tools = [
            Tool(name="search_products", handler=self.search_products),
            Tool(name="create_order", handler=self.create_order),
            Tool(name="get_inventory", handler=self.get_inventory),
        ]
    
    def search_products(self, query: str, category: str = None) -> List[dict]:
        # 实现产品搜索逻辑
        pass
```

## SDK与实现

### Python SDK

```python
# 安装SDK
# pip install mcp

from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("example-server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="get_weather",
            description="Get current weather for a location",
            inputSchema={
                "type": "object",
                "properties": {
                    "location": {"type": "string"}
                }
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "get_weather":
        return [TextContent(type="text", text=f"Weather in {arguments['location']}: Sunny")]
    raise ValueError(f"Unknown tool: {name}")
```

### TypeScript SDK

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "example-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_product",
      description: "Retrieve a product by its ID",
      inputSchema: {
        type: "object",
        properties: {
          id: { type: "string" }
        }
      }
    }
  ]
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_product") {
    return { content: [{ type: "text", text: "Product details..." }] };
  }
  throw new Error("Unknown tool");
});
```

## 生态系统

### 支持MCP的平台

截至2026年初，MCP已被广泛采用：

| 平台 | 支持情况 |
|------|----------|
| **Claude (Anthropic)** | 原生MCP客户端 |
| **Claude Code** | 使用MCP进行工具集成 |
| **VS Code** | Agent模式支持MCP工具 |
| **OpenAI** | Agents SDK支持MCP |
| **GitHub Copilot** | Agent Mode支持 |
| **LangChain** | MCP服务器集成 |
| **Azure** | AI Agent服务集成 |

### 官方服务器注册表

Anthropic维护着官方MCP服务器注册表：[github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)

涵盖：
- 文件系统操作
- Git操作
- 数据库连接
- Slack/Discord集成
- 浏览器自动化

## 安全模型

### OAuth 2.1集成

MCP规范包含受保护资源的OAuth 2.1流程：

```json
{
  "scopes": ["read", "write"],
  "authentication": {
    "type": "oauth",
    "clientId": "mcp-server-id",
    "authorizationUrl": "https://auth.example.com/authorize",
    "tokenUrl": "https://auth.example.com/token"
  }
}
```

### 安全最佳实践

1. **最小权限原则**：每个MCP服务器只暴露必要的工具
2. **输入验证**：严格验证所有工具参数
3. **沙箱执行**：敏感操作在沙箱中执行
4. **审计日志**：记录所有工具调用

## MCP与AI Agent工作流

### 完整工作流

```
用户查询 → MCP Client → list_tools() → 发现可用工具
                                    ↓
                              工具元数据
                                    ↓
                              提示工程
                                    ↓
                            call_tool() → 执行
                                    ↓
                              返回结果
                                    ↓
                              代理决策
```

### 多步骤代理示例

```python
async def agent_workflow(user_query: str):
    # 1. 发现工具
    tools = await mcp_client.list_tools()
    
    # 2. 选择工具
    selected_tool = await select_tool(tools, user_query)
    
    # 3. 生成参数
    arguments = await generate_arguments(selected_tool, user_query)
    
    # 4. 执行
    result = await mcp_client.call_tool(selected_tool.name, arguments)
    
    # 5. 响应用户
    return format_response(result)
```

## 参考资料

[^1]: Model Context Protocol Specification - https://modelcontextprotocol.info/specification/

[^2]: MCP Architecture for AI Agents - https://www.web4agents.org/en/docs/mcp

[^3]: OpenAI Apps SDK with MCP - https://developers.openai.com/apps-sdk/concepts/mcp

[^4]: VS Code MCP Developer Guide - https://code.visualstudio.com/api/extension-guides/mcp
