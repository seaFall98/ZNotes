MCP 的 **HTTP+SSE** 主要缺陷在于：它是早期远程传输方案，已经被 **Streamable HTTP** 取代。官方规范在 2025-03-26 版本中明确写到，Streamable HTTP “replaces” 2024-11-05 的 HTTP+SSE transport；后续草案也将 HTTP+SSE 归入 Deprecated，并建议迁移到 Streamable HTTP。([modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports "Transports - Model Context Protocol"))
[[Streamable HTTP详解]]
## 主要缺陷

### 1. 两条通道，架构复杂

旧 HTTP+SSE 通常需要：

- 一条 **SSE 长连接**：server → client
    
- 一个 **HTTP POST endpoint**：client → server
    

这会让连接管理、会话绑定、反向代理配置、鉴权、负载均衡都更复杂。相比之下，Streamable HTTP 使用单一 MCP endpoint，同时支持 POST/GET，并可按需使用 SSE 流式返回。([modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports "Transports - Model Context Protocol"))

## 2. 对基础设施不友好

SSE 是长连接。实际部署时常遇到：

- 代理或网关超时断开；
    
- serverless / edge 平台对长连接支持不一致；
    
- 连接保活、重连、心跳、缓冲策略需要额外处理；
    
- 横向扩容时 session affinity 更麻烦。
    

新规范里 Streamable HTTP 明确考虑了断线、重连、`Last-Event-ID`、可恢复流、session ID 等机制，说明旧方案在生产环境下确实容易暴露这些问题。([modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

## 3. 客户端实现负担较重

HTTP+SSE 客户端要同时处理：

- SSE 事件流；
    
- 普通 HTTP POST；
    
- 两个 endpoint 的生命周期；
    
- 断线重连；
    
- 消息顺序和请求关联；
    
- server-to-client notification / request 的路由。
    

Streamable HTTP 把交互收敛到一个 MCP endpoint，客户端只需要先尝试 POST 初始化；如果失败，再按 backward compatibility 逻辑判断是否是旧 HTTP+SSE server。([modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports "Transports - Model Context Protocol"))

## 4. 安全边界更容易出错

远程 MCP server 本身就需要认真处理安全问题。官方规范特别强调：

- 必须校验 `Origin`，防 DNS rebinding；
    
- 本地 server 应只绑定 `127.0.0.1`；
    
- 应实现适当认证。
    

HTTP+SSE 因为有独立 SSE endpoint 和 POST endpoint，常见风险是其中一个 endpoint 漏掉鉴权、Origin 校验、CORS 限制或 session 校验。Streamable HTTP 并不会自动解决安全问题，但统一入口更利于集中治理。([modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

## 5. 状态与恢复语义不够清晰

SSE 断开不等于请求取消。新规范明确要求：断线不应被解释为 client cancel；取消应通过 MCP `CancelledNotification` 表达，并建议 server 让 stream 可恢复。([modelcontextprotocol.io](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports "Transports - Model Context Protocol"))

旧 HTTP+SSE 在实际实现中容易出现：

- 连接断了但任务还在跑；
    
- client 重连后不知道之前请求状态；
    
- server 无法准确关联重连请求；
    
- 重复执行 tool call；
    
- 响应丢失或乱序。
    

## 6. 生态正在迁移，兼容性会越来越差

Cloudflare 文档也说明，SSE 以前用于远程 MCP 连接，但已被 Streamable HTTP 取代；新的远程 MCP server 推荐使用 Streamable HTTP。([Cloudflare Docs](https://developers.cloudflare.com/agents/model-context-protocol/protocol/transport/ "Transport · Cloudflare Agents docs"))

这意味着继续做 HTTP+SSE 会有几个现实问题：

|问题|后果|
|---|---|
|新客户端优先支持 Streamable HTTP|老 server 可能需要兼容层|
|新 SDK / hosting 平台减少 SSE 路径支持|运维成本上升|
|规范状态为 Deprecated|长期技术债明确|
|双 endpoint 兼容逻辑复杂|测试矩阵扩大|

## 结论

**HTTP+SSE 的核心问题不是 SSE 本身不能用，而是它作为 MCP 远程传输方案过于笨重：双通道、长连接、状态恢复、安全治理和部署兼容性都更复杂。**

新项目应直接用 **Streamable HTTP**。只有两种情况才建议保留 HTTP+SSE：

1. 需要兼容老 MCP client；
    
2. 你的现有 server 已部署 HTTP+SSE，短期不能迁移。
    

更稳妥的生产策略是：**新 endpoint 用 Streamable HTTP，旧 HTTP+SSE 只作为临时兼容层，并设定下线窗口。**