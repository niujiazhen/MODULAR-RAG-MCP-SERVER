# MCP Server 协议 - D5.1 MCP协议概述

> 学习日期: 2026-03-24
> 综合评分: 8.5/10 | 追问轮数: 1/1

---

**Q1: 本项目的 MCP Server 基于 JSON-RPC 2.0 协议通过 stdio 传输与客户端通信。请描述一次完整 MCP 会话从建立到工具调用经历的协议交互步骤，以及为什么项目要专门做 `_redirect_all_loggers_to_stderr()`。**

**标准答案**: 一次完整 MCP 会话经历五个阶段：

1. **传输层建立（Transport Setup）**：客户端（如 Claude Desktop）以子进程方式启动 MCP Server，通过 stdio 建立双向通信管道。`server.py:105` 中 `async with mcp.server.stdio.stdio_server() as (read_stream, write_stream)` 创建传输层。stdout 作为 Server→Client 通道，stdin 作为 Client→Server 通道。

2. **初始化握手（Initialize Handshake）**：三步握手——① Client 发送 `initialize` 请求（携带 protocolVersion、capabilities、clientInfo）；② Server 返回 `initialize` 响应（声明 capabilities、serverInfo 和协商后的 protocolVersion），项目中 `protocol_handler.py:181-189` 的 `get_capabilities()` 返回 `{"tools": {}}` 声明支持工具调用；③ Client 发送 `initialized` 通知（notification，无 id 字段，不需要响应），确认握手完成。

3. **工具发现（Tool Discovery）**：Client 发送 `tools/list` 请求，Server 通过 `handle_list_tools()`（protocol_handler.py:246-249）调用 `get_tool_schemas()` 返回所有注册工具的 name、description、inputSchema。

4. **工具调用（Tool Call）**：Client 发送 `tools/call` 请求（包含 name 和 arguments），Server 通过 `handle_call_tool()`（protocol_handler.py:252-257）执行工具逻辑，返回 `CallToolResult`（包含 content 数组和 isError 标志）。

5. **会话终止（Shutdown）**：客户端关闭 stdin 管道或终止子进程，Server 检测到 read_stream EOF 后退出 `server.run()` 循环，执行清理。

整个过程中，每条消息都是严格的 JSON-RPC 2.0 格式：request 有 `id` + `method` + `params`，response 有对应 `id` + `result`/`error`，notification 有 `method` 但无 `id`。

**关于 `_redirect_all_loggers_to_stderr()`**：stdio transport 将 stdout 独占为协议通道，stdout 上的每个字节都必须是合法的 JSON-RPC 消息。如果 logging 或 print() 向 stdout 输出日志文本，会混入 JSON-RPC 流中导致：协议损坏（客户端解析遇到非 JSON 内容触发 Parse Error -32700，会话断开）和消息粘连（日志文本粘在 JSON 消息前后导致无法正确分帧）。因此必须将所有 logger handler 替换为指向 stderr 的 handler。

---

**Q2: `create_mcp_server()` 中用 `@server.list_tools()` 和 `@server.call_tool()` 两个装饰器将 `ProtocolHandler` 桥接到 MCP SDK 的 `Server` 上。这种"自建 ProtocolHandler + SDK 装饰器桥接"的分层设计，相比直接让每个 tool 文件自己注册到 SDK Server 上，有什么架构优势？**

**标准答案**: ProtocolHandler 本质上是一个反腐败层（Anti-Corruption Layer），将 SDK 协议机制和工具业务逻辑隔离，有四大架构优势：

1. **SDK 解耦**：工具只认识 `ProtocolHandler.register_tool()` 接口，不依赖 SDK 的 Server 类。SDK 升级或替换时只改 `create_mcp_server()` 一个桥接点，tool 文件零改动。

2. **可测试性**：ProtocolHandler 是纯 dataclass，直接实例化即可测试工具注册和执行，无需启动真实 stdio server 走协议流程。

3. **集中式错误处理**：`execute_tool()` 统一捕获 `TypeError` → 参数错误、`Exception` → 内部错误且不泄露堆栈。工具不用各自重复写防御逻辑。此外还有**返回类型归一化**：处理 `CallToolResult`、`str`、`list` 和兜底 `str()` 四种返回类型，使 tool handler 不必关心协议输出格式。

4. **单一注册入口**：`_register_default_tools()` 一个函数列出所有工具，可控顺序、条件注册、去重校验（`register_tool()` 中 `if name in self.tools: raise ValueError` 防止重名）。不用 grep 全库才知道注册了什么。

补充细节：`create_mcp_server()` 还通过 `server._protocol_handler = protocol_handler` 存储引用，配套 `get_protocol_handler()` 访问器提供"回溯通道"，使外部代码可以从 Server 实例反查到 ProtocolHandler。

---

## 📊 评价报告

**知识域**: MCP Server 协议 > **知识点**: D5.1 MCP 协议概述 — MCP 会话交互流程与 stdio 协议约束
**追问轮数**: 1/1

### ✅ 回答亮点
- 五阶段协议流程描述完整且精确，特别是 Initialize 三步握手中 `initialized` notification 无 id 的细节
- 代码引用准确到行号（server.py:105, protocol_handler.py:181-189, 246-249, 252-257）
- stderr 重定向的因果链清晰：stdout 独占 → 日志混入 → Parse Error -32700 → 会话断开
- 追问中"反腐败层"概念精准概括了 ProtocolHandler 的架构角色，四个优势维度全面且言简意赅

### ⚠️ 需要加强
- `execute_tool()` 的返回类型归一化未提及：处理了 CallToolResult、str、list 和兜底 str() 四种返回类型
- `register_tool()` 的重名校验（`if name in self.tools: raise ValueError`）未展开
- `server._protocol_handler` 的存储模式及 `get_protocol_handler()` 访问器未提及

### 📈 评分明细

| 维度 | 分数 | 说明 |
|------|------|------|
| 准确性 | 9/10 | 所有协议阶段和代码引用均准确无误 |
| 深度 | 8.5/10 | 主答协议流程深入到 notification 语义层；追问缺少返回类型归一化等实现细节 |
| 代码关联 | 9/10 | 多处精确行号引用，追问中引用了具体方法名和异常映射 |
| 设计思维 | 9/10 | 反腐败层框架体现成熟架构认知，四维度分析结构清晰 |

### 🏆 综合评分: 8.5/10

### 📊 学习进度: 13/45 知识点已掌握

---

## 📚 学习指南

### 📂 相关代码
- [server.py:82-113](src/mcp_server/server.py#L82-L113) — `run_stdio_server_async()` 完整启动流程
- [protocol_handler.py:42-58](src/mcp_server/protocol_handler.py#L42-L58) — `ProtocolHandler` dataclass 定义
- [protocol_handler.py:108-179](src/mcp_server/protocol_handler.py#L108-L179) — `execute_tool()` 四种返回类型归一化 + 三层异常捕获
- [protocol_handler.py:211-262](src/mcp_server/protocol_handler.py#L211-L262) — `create_mcp_server()` 桥接工厂
- [server.py:47-79](src/mcp_server/server.py#L47-L79) — `_preload_heavy_imports()` 防止 import lock 死锁

### 📄 相关文档
- [DEV_SPEC.md](DEV_SPEC.md) — "MCP 生态集成" 章节

### 🔗 参考资料
- **JSON-RPC 2.0 Specification** — request/response/notification 消息格式的协议基础
- **MCP Protocol Specification** — initialize/initialized 握手、tools/list、tools/call 标准定义
- **Anti-Corruption Layer (DDD)** — 反腐败层概念，用于隔离外部系统接口

### 💡 建议学习路径
1. 先通读 [protocol_handler.py](src/mcp_server/protocol_handler.py) 重点看 `execute_tool()` 的四种返回类型处理
2. 再看 [tools/query_knowledge_hub.py](src/mcp_server/tools/query_knowledge_hub.py) 理解具体 tool 如何注册
3. 运行 `python -m src.mcp_server.server` 观察 stderr 日志输出
4. 尝试在 `get_capabilities()` 中添加 `"resources": {}` 能力声明，理解能力协商扩展方式
