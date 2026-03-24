# MCP Server 协议 - D5.3 ProtocolHandler

> 学习日期: 2026-03-24
> 综合评分: 8.5/10 | 追问轮数: 1/1

---

**Q1: 当 MCP 客户端发送一个 `tools/call` 请求时，请求从进入 MCP SDK 到最终返回结果，经过了哪些层次的路由和分发？请以 `query_knowledge_hub` 为例，描述完整的调用链路，并重点分析 `execute_tool()` 方法内部的三层异常处理。**

**标准答案**: 以 `query_knowledge_hub` 为例，完整调用链路经过四层：

1. **SDK 路由层**：客户端 JSON-RPC 请求经 stdin 进入 `Server.run()`，SDK 按 `method` 字段路由到对应的注册处理器。

2. **装饰器桥接层**：`@server.call_tool()` 装饰器注册的 `handle_call_tool(name, arguments)` 函数被调用，它委托给 `protocol_handler.execute_tool(name, arguments)`。

3. **ProtocolHandler 分发层**：`execute_tool()` 在 `self.tools` 字典中查找 `name` 对应的 `ToolDefinition`，通过 `tool.handler(**arguments)` 调用 handler。`**arguments` 展开意味着 handler 的函数签名就是参数契约，参数不匹配时 Python 自动抛出 `TypeError`。

4. **Tool Handler 业务层**：`query_knowledge_hub_handler()` 获取单例 Tool 实例，调用 `tool.execute()` → `asyncio.to_thread()` 执行阻塞 I/O，结果封装为 `CallToolResult` 原路返回写入 stdout。

**三层异常处理**（protocol_handler.py:122-179）：

1. **工具不存在**（`name not in self.tools`）：客户端调了错误的工具名，`logger.warning` 记录，返回明确提示。
2. **TypeError**：客户端传参与 handler 签名不匹配（多了/少了/类型错），`logger.error` 记录，返回具体错误帮客户端修正。
3. **Exception 兜底**：服务端内部故障（如 ChromaDB 崩溃），`logger.exception` 记录完整堆栈用于调试，但只返回泛化错误信息 `"Internal server error"` 不泄露实现细节。

设计原则：客户端的错给诊断信息，服务端的错藏实现细节，任何异常都不逃逸到 SDK 层。日志级别从 warning → error → exception 递进，对应严重程度从"可恢复的客户端错误"到"需要排查的服务端故障"。

补充：`JSONRPCErrorCodes` 类（protocol_handler.py:21-28）定义了标准 JSON-RPC 错误码（PARSE_ERROR -32700 到 INTERNAL_ERROR -32603），但 `execute_tool()` 实际上未使用这些常量——它通过 MCP SDK 的 `CallToolResult(isError=True)` 返回错误，而非 JSON-RPC 标准的 error response。这是因为 MCP 协议在 `tools/call` 层面用 `isError` 标志区分成功/失败，而非底层 JSON-RPC error。

---

**Q2: `execute_tool()` 在 handler 成功执行后，对返回值做了四种类型的归一化处理。请列出这四种情况，并分析为什么不直接要求所有 handler 统一返回 `CallToolResult`。**

**标准答案**: 四种返回值归一化（protocol_handler.py:136-154）：

1. **`CallToolResult`** — 直接返回，不做处理。适用于需要精确控制响应结构的复杂工具。
2. **`str`** — 包装为 `TextContent` 放入 `CallToolResult`。适用于只需返回文本的轻量工具。
3. **`list`** — 视为 content 列表直接放入 `CallToolResult`。适用于多模态响应（如 `TextContent` + `ImageContent` 混合），如 `query_knowledge_hub` 的 `to_mcp_content()` 返回的多内容块。
4. **其他类型** — `str()` 转字符串后包装为 `TextContent`。兜底处理，确保任何返回值都能正确序列化。

不强制统一返回 `CallToolResult` 的原因：降低 tool handler 的编写门槛。轻量工具（如 `list_collections`）只需返回格式化字符串，强制构造 `CallToolResult(content=[TextContent(type="text", text=...)])` 是不必要的样板代码。归一化逻辑集中在 `ProtocolHandler` 一处，handler 只关心业务逻辑，返回最自然的类型即可。本质上是把协议适配的职责从 N 个 tool 收拢到 1 个 ProtocolHandler，符合单一职责原则。

---

## 📊 评价报告

**知识域**: MCP Server 协议 > **知识点**: D5.3 ProtocolHandler — 请求路由、分发与异常处理
**追问轮数**: 1/1

### ✅ 回答亮点
- 四层调用链路描述精准完整
- 三层异常处理的区分原则精炼
- 四种返回类型归一化全部正确列出
- "N→1 收拢协议适配"的设计本质抓得精准

### ⚠️ 需要加强
- JSONRPCErrorCodes 定义了但未使用的设计选择未提及
- 三层异常的日志级别递进（warning → error → exception）未提及
- `**arguments` 展开作为参数契约机制、TypeError 触发原因未展开
- list 返回类型支持多模态响应的实际用途未提及

### 📈 评分明细

| 维度 | 分数 | 说明 |
|------|------|------|
| 准确性 | 9/10 | 路由链路和归一化逻辑全部准确 |
| 深度 | 8.5/10 | 异常处理原则深入，但 JSONRPCErrorCodes 未触及 |
| 代码关联 | 7/10 | 流程描述准确但缺少具体代码行号引用 |
| 设计思维 | 9/10 | "N→1 收拢" 和异常隔离原则展现出色架构思维 |

### 🏆 综合评分: 8.5/10

### 📊 学习进度: 15/45 知识点已掌握

---

## 📚 学习指南

### 📂 相关代码
- [protocol_handler.py:20-28](src/mcp_server/protocol_handler.py#L20-L28) — JSONRPCErrorCodes 定义
- [protocol_handler.py:108-179](src/mcp_server/protocol_handler.py#L108-L179) — execute_tool() 完整实现
- [protocol_handler.py:136-154](src/mcp_server/protocol_handler.py#L136-L154) — 四种返回类型归一化
- [protocol_handler.py:156-179](src/mcp_server/protocol_handler.py#L156-L179) — 三层异常处理
- [protocol_handler.py:246-257](src/mcp_server/protocol_handler.py#L246-L257) — 装饰器桥接

### 🔗 参考资料
- **JSON-RPC 2.0 Error Codes** — 标准错误码 vs MCP isError 标志的设计取舍
- **Python \*\*kwargs** — handler(**arguments) 展开与 TypeError 触发机制

### 💡 建议学习路径
1. 重读 protocol_handler.py:108-179，逐行跟踪 happy path 和三条 error path
2. 对比 JSONRPCErrorCodes 定义与实际使用，理解 MCP CallToolResult(isError=True) 和 JSON-RPC error response 的区别
3. 查看 query_knowledge_hub.py to_mcp_content() 返回 list 的多模态场景
