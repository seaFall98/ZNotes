
MCP支持两种主要的传输方式，分别适用于不同场景。

本地场景用stdio，Client把Server作为子进程启动，通过标准输入输出通信，延迟极低，不用开端口，也没有网络安全问题，我用ClaudeDesktop接本地工具走的就是这种方式。

远程场景现在推荐用StreamableHTTP，Server作为独立的HTTP服务部署，多个Client可以共享同一个Server，适合团队统一管理工具服务。

MCP早期版本（2024-11-05规范）的远程传输是  HTTP+SSE」双端点方案，2025年3月的规范更新里被标记为deprecated（保留向后兼容但不推荐新项目使用），StreamableHTTP成为了推荐的远程传输方式。

不管哪种传输方式，底层消息格式都统一用JSON-RPC2.0，传输方式只影响「怎么传」，消息协议本身不变。