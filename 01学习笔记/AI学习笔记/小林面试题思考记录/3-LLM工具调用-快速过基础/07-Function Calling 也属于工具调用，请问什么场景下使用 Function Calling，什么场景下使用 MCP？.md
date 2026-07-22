
依旧可以联系AI MCP Gateway的作用

---
## 简答

Function Calling 和 MCP 不是二选一，而是两个层次的问题。Function Calling 解决的是“模型如何向当前应用请求执行一个函数”；MCP 解决的是“应用如何标准化地连接和复用外部工具、资源和提示”。

如果只是一个应用里的少量内部函数，直接使用 Function Calling 就够了，简单、可控、延迟低。

Function Calling 的适用场景。快速原型、工具只为单一应用服务、需要精细控制执行逻辑、部署环境受限这四种场景

只要工具需要跨项目或跨团队复用、或者数量多了管理麻烦、或者社区已经有现成的 MCP Server 可以直接配置，MCP 就值得上了

判断的核心问题只有一个：这个工具会不会在这个应用之外被用到？会的话，把它封装成 MCP Server 是更长远的选择。

做 Agent 系统的话更应该选 MCP，工具来源多、数量大

实际生产中经常组合使用：应用通过 MCP 接入外部工具，再把这些工具转换成模型可以调用的 Function Calling。简单记忆就是：**Function Calling 解决模型怎么调用，MCP 解决应用怎么接入和复用。**

参考：[OpenAI Function Calling](https://developers.openai.com/api/docs/guides/function-calling)、[MCP Architecture](https://modelcontextprotocol.io/specification/latest/architecture)

---

