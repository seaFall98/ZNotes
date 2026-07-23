
langraph 图 节点 条件边 state共享 handoff

deepagent：
[deepagent](https://my.feishu.cn/docx/WQVndx4S0odEFUxNXGKcmZSYn0c)
专门的SubAgentMiddleware
task工具用yaml创建subagent  上下文隔离 子代理工作的最终返回的是一条toolmessage

专门的agent protocol  server
subagent 
asyncsubagent不阻塞主agent

在 DeepAgent 里，主代理（Supervisor）与异步子代理（Async Subagent）通信，主要通过 **5 个核心方法（工具）** ：

*   **启动 (`start_async_task`)**: 启动一个后台任务，并立即返回一个任务 ID，主代理不会阻塞。
*   **查看状态 (`check_async_task`)**: 获取某个任务的当前状态和结果（如果已完成）。
*   **插入新指令 (`update_async_task`)**: 向一个正在运行的任务发送新的指令，它会中断当前运行并用新指令重启任务。
*   **停止 (`cancel_async_task`)**: 停止一个正在运行的任务。
*   **列出任务 (`list_async_tasks`)**: 列出所有被跟踪任务的实时状态摘要。
用户可以主动提出这些要求，主agent就会调用这些工具


---
多agent还可能有A2A,但是尽量避免A2A